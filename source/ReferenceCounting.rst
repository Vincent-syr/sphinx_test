引用计数
######################
对象生命周期管理一直是C/C++程序中比较困难且十分重要的一部分。本文主要介绍\
ReferenceCounting 技术的应用。

简单的对象生命周期管理
======================
简单的对象生命周期，仅有一个所有者（其所有者可能转移，但仅会有一个）：

1. **Global object**: Its owner is the program or the DSO. It's destroied
   when program exit or the DSO is unloaded.

   .. code-block:: c++
    :linenos:
    :emphasize-lines: 2

    class GlobalObject {};
    static GlobalObject globalObject;
  
  The ``static`` keyword is not important here and just indicates its visibility
  to the other compilation units. The ``globalObject`` is constructed before the ``main``
  and destroied then program exit(``atexit``) or when the DSO is unloaded if the it is
  compiled into DSO.

2. **Local static object**: Its owner is the program or the DSO. It's
   destoried as same as **Global object**.

   .. code-block:: c++
    :linenos:
    :emphasize-lines: 3

    class Object {};
    Object* ObjectInstance(void) {
      static Object sInst;
      return &sInst;
    }

  The ``static`` is mandatory, it indicates ``sInst`` has static storage. The ``sInst``
  is constructed when the program first try to access it. This is thread safe, and it
  is ensured by C++ once time construction mechanism. And it is destroied when program exit
  or then DSO is unloaded if it is compiled into DSO.

3. **Thread local object**: Its owner is the calling thread. It's destroied
   when the thread exit.

   .. code-block:: c++
    :linenos:
    :emphasize-lines: 3

    class Object {};
    Object* GetCurrentObject(void) {
      static thread_local Object stInst;
      return &stInst;
    }

   The ``static`` keyword is mandatory, it indicates ``stInst`` has static storage.
   The ``thread_local`` keyword indicates ``stInst`` has thread local storage. The
   ``stInst`` is constructed when a thread first try to access it. And it destoried
   then the thread exit.

   **Notice**: ``thread_local`` use ``__cxa_thread_atexit(dso, exit_fn)`` to
   register destructor of ``stInst`` if it is not trivially destructible and this
   call also causes DSO's reference increment.

4. **Common Object**: Owned by another object and is destroied when its owner
   is destroied.
   
   .. code-block:: c++
    :linenos:
    :emphasize-lines: 4, 8

    class Object{};
    class AnotherObjectDirect {
    private:
      Object object_;
    };
    class AnotherObjectPtr {
     private:
      Object* object_;
     public:
      AnotherObjectPtr()
        : object_(new Object) {}

      ~AnotherObjectPtr() {
        delete object_;
      }
    };
  
  Both ``AnotherObjectDirect`` and ``AnotherObjectPtr`` destroy ``object_`` when
  it is destroied.

*DSO*: Dynamic Shared Object(Dynamic Loaded Library)

复杂对象生命周期管理
=====================
对于复杂的场景需要更为复杂的管理方法：

1. 对象作为一项资源会被多个地方所引用，而各自释放其引用的时间是不确定的，当引用没有释放\
   时该对象必须处于有效状态。
2. 当对象将要销毁时，其它地方可能临时访问一下该对象，并尝试获得其引用（尝试意味着可能失败，\
   意味着未实际销毁的对象还能临时访问一下）。

**在这两种场景下，引用计数都是一个良好的解决方案。**

针对第一种场景:
~~~~~~~~~~~~~~~~~~~~~

基本定义
---------------------

.. code-block:: c++
  :linenos:
  :emphasize-lines: 4, 10
  :caption: Object Creation

  #include <atomic>

  struct object {
    std::atomic<int> count;
    int x, y, z;
  };
  struct object* object_create(...) {
    struct object* obj = (struct object*)malloc(sizeof(*obj));
    if (!obj) { /* handle ENOMEM */ }
    obj->count.load(1, std::memory_order_relaxed);
    return obj;
  }

上述代码中， ``object`` 有一个 ``count`` 成员表示其引用计数。考虑到线程安全，针对\
``count`` 的相关操作都需要使用原子操作。``line#10`` 对 ``count`` 的初始化也是使用\
``RELAXED`` 语义，如果创建的对象后续需要传递给其它线程，通常还会有其它 *memory order*\
相关的操作会执行 ``RELEASE + ACQUIRE``，这样在跨线程传递对象就是安全的。例如：

.. code-block:: c++
  :linenos:
  :emphasize-lines: 5, 12, 16

  static object* volatile objp = nullptr;

  // thread0:
  object* tmp = object_create(...);
  std::atomic_thread_fence(std::memory_order_release);
  objp = tmp;

  /**************************************************/

  // thread1:
  object* tmp = objp;
  std::atomic_thread_fence(std::memory_order_acquire);
  while (!tmp) {
    cpu_relaxed();
    tmp = objp;
    std::atomic_thread_fence(std::memory_order_acquire);
  }

由于高亮的代码增加的 *RELEASE+ACQUIRE* 语义，当thread1读到 ``objp`` 不是\
``nullptr`` 时，thread1也能看到thread0做的相关修改。对齐的 *pointer* 的 ``load`` \
``store`` 是原子的，因此这里不用额外做其它操作。

获得引用
------------------

.. code-block:: c++
  :linenos:
  :emphasize-lines: 2
  :caption: Obtain Object Reference

  struct object object_get(struct object* obj) {
    obj->count.fetch_add(1, std::memory_order_relaxed);
    return obj;
  }

注意，上述代码中对 ``count`` 加 ``1`` 使用了 ``std::memory_order_relaxed``，因为这里\
``obj`` 与 ``count`` 并没有数据依赖关系。同前面提到的一样，如果该 **pointer** 需要跨\
线程传递，通常还需要其它的同步操作来执行 *ACQUIRE+RELEASE* 语义。

释放引用
------------------

.. code-block:: c++
  :linenos:
  :emphasize-lines: 2,4

  void object_put(struct object* obj) {
    int ret = obj->count.fetch_sub_explicit(1, std::memory_order_release);
    if (ret == 1) {
      std::atomic_signal_fence(std::memory_order_acquire);
      free(obj);
    }
  }

注意，上述代码中有两个重点：

1. ``fetch_sub_explicit`` 应用了 ``std::memory_order_release``， 这是因为\
当执行 ``object_put`` 时，在 ``object_put`` 之前的所有写操作必须对其它线程可见尤其是\
最后一个调用 ``object_put`` 的线程（这些写操作必须在 ``object_put`` 之前执行）。
2. 在 ``ret == 1`` 时，也就是当前是最后一个引用时，首先执行了
``std::atomic_thread_fence(std::memory_order_acquire)``，这个 ``ACQUIRE`` 保证了\
其它线程的在 ``object_put`` 之前的操作对当前线程可见（已经完成）。

针对第二种场景
~~~~~~~~~~~~~~~~~
通常在这种场景下，我们会有一个全局的容器，将所有的对象放在该容器中，当需要访问其中的对象时\
在从容器中查找。

基本定义
-----------------

.. code-block:: c++
  :linenos:
  :caption: Object Declaration

  #include <map>
  #include <mutex>

  struct Object {
    std::atomic<int> count;
    std::string name;
    int x, y, z;

    Object(const std::string& _name)
      : count(1)
      , name(_name)
    {}
  };
  Object* CreateObject(const std::string& name) {
    return new Object(name);
  }

  std::mutex objMapMutex;
  std::map<std::string, Object*> objMap;

获取一个对象
-----------------

.. code-block:: c++
  :linenos:

  Object* ObjectGet(const std::string& key) {
    std::lock_guard<std::mutex> guard(objMapMutex);
    auto iter = objMap.find(key);
    if (iter == objMap.end()) {
      Object* obj = CreateObject(key);
      objMap.emplace(key, obj);
      return obj;
    } else {
      Object* obj = iter->second;
      obj->count.fetch_add(1, std::memory_order_relaxed);
      return obj;
    }
  }

释放一个对象
-----------------

.. code-block:: c++
  :linenos:
  :emphasize-lines: 2,4
  :caption: Release Object(buggy)

  void ObjectPut(Object* obj) {
    int ret = obj->count.fetch_sub(1, std::memory_order_release);
    if (ret == 1) {
      std::atomic_thread_fence(std::memory_order_acquire);
      objMapMutex.lock();
      auto iter = objMap.find(obj->name);
      if (iter != objMap.end()) {
        objMap.erase(iter);
      }
      objMapMutex.unlock();
      delete obj; // 最好可移到 objMapMutex 临界区之外。
    }
  }

In the code above, line #4 can be eliminated, because the lock of ``mutex`` contains
a ``ACQUIRE`` semantics.

容易看出， ``ObjectGet`` 与 ``ObjectPut`` 一起使用时，会有一个 ``bug``，

  当一个线程执行 ``ObjectPut`` 并成功将 ``obj`` 的引用计数降至零，并且在执行 line #5 之前
  另外一个线程执行 ``ObjectGet`` 并成功将 ``obj`` 的引用计数增加到1，然后返回。显然在 ``ObjectPut``
  返回之后，执行 ``ObjectGet`` 的线程，将获得一个无效的对象。

解决方案一，将 ``ObjectPut`` 中的 line#5 提到 line#2 之前：

.. code-block:: c++
  :linenos:
  :caption: 解决方案一

  void ObjectPut1(Object* obj) {
    objMapMutex.lock();
    int ret = obj->count.fetch_sub(1, std::memory_order_release);
    if (ret == 1) {
      std::atomic_thread_fence(std::memory_order_acquire);
      auto iter = objMap.find(obj->name);
      if (iter != objMap.end()) {
        objMap.erase(iter);
      }
      objMapMutex.unlock();
      delete obj; // 最好可移到 objMapMutex 临界区之外。
    } else {
      objMapMutex.unlock();
    }
  }

在这种方案中，所有的 ``sub``, ``add`` 都被 ``objMapMutex`` 所保护，因此不会有问题。但是这种\
方案的缺陷也是明显的，每一次释放引用计数都需要获得锁，这个会导致严重的锁竞争。

解决方案二，修改 ``ObjectGet`` 当且仅当 ``obj`` 引用计数大于 ``0`` 时才能成功获取：

.. code-block:: c++
  :linenos:
  :caption: 解决方案二

  bool atomic_add_unless_zero(std::atomic<int>* count, int add) {
    int val = count->load(std::memory_order_relaxed);
    do {
      if (val == 0) {
        return false;
      }
    } while (count->compare_exchange_weak(val, val + add));
    return true;
  }

  Object* ObjectGet1(const std::string& key) {
    std::lock_guard<std::mutex> guard(objMapMutex);
    auto iter = objMap.find(key);
    if (iter != objMap.end()) {
      Object* obj = iter->second;
      if (atomic_add_unless_zero(&obj->count, 1)) {
        return obj;
      }
      return nullptr; //! Object is deleting
    } else {
      Object* obj = CreateObject(key);
      objMap.emplace(key, obj);
      return obj;
    }
  }

在这种方案中， ``ObjectPut`` 仅当释放最后一个引用计数时才需要获取锁。减少了锁竞争，但其\
缺点是 ``ObjectGet`` 中 line#19 不得不返回 ``nullptr`` （当然可以再通过一些重试的办\
法来部分解决这个问题）。

另外， ``CreateObject`` 被放在 ``objMapMutex`` 的保护下执行，也就是多个对象的创建此时\
串行的。如果对对象的创建有性能要求，在进行分析后可以做以下改动，将 ``CreateObject`` 拆\
成两部分。将耗时长的部分放在 ``objMapMutex`` 之外执行，然后采用通知机制。

改进方案一
------------------

.. code-block:: c++
  :linenos:

  #include <atomic>
  #include <mutex>
  #include <condition_variable>

  enum ObjectState {
    ObjectState_New,
    ObjectState_Initialized,
  };
  struct Object {
    Object(const std::string& _name)
      : count(1)
      , name(_name)
      , state(ObjectState_New)
    {}

    std::atomic<int> count;
    std::string name;
    int x, y, z;

    ObjectState state;
    std::condition_variable waitState;
    std::mutex stateMutex;
  };

  Object* CreateObject(const std::string& name) {
    return new Object(name);
  }

  void ObjectInitMayCostLongTime(Object* obj) {
    // long time cost operations
    std::lock_guard<std::mutex> guard(obj->stateMutex);
    obj->state = ObjectState_Initialized;
    obj->waitState.notify_all();
  }
  
  void ObjectWaitForInitDone(Object* obj) {
    std::unique_lock<std::mutex> lock(obj->stateMutex);
    while (obj->state == ObjectState_New) {
      obj->waitState.wait(lock);
    }
  }

获取引用
--------------
.. code-block:: c++
  :linenos:
  
  Object* ObjectGet2(const std::string& key) {
    std::lock_guard<std::mutex> guard(objMapMutex);
    auto iter = objMap.find(key);
    if (iter != objMap.end()) {
      Object* obj = iter->second;
      if (atomic_add_unless_zero(&obj->count, 1)) {
        guard.unlock();
        ObjectWaitForInitDone(obj);
        return obj;
      }
      return nullptr; //! Object is deleting
    } else {
      Object* obj = CreateObject(key);
      objMap.emplace(key, obj);
      guard.unlock();
      ObjectInitMayCostLongTime(obj);
      return obj;
    }
  }


在实践也可以将 ``ObjectPut1`` 与 ``ObjectGet2``中的 ``LongInit + Wait`` 结合。\
当使用 ``ObjectPut1`` 时， ``ObjectGet2`` 就不存在返回 ``nullptr`` 这个分支了。

-----------------------------------

**附：**

容易误导的方案，双重检查，在 `ObjectPut` 中执行双重检查

.. code-block:: c++
  :linenos:
  :caption: 容易误导的方案

  void ObjectPut(Object* obj) {
    int ret = obj->count.fetch_sub(1, std::memory_order_release);
    if (ret == 1) {
      std::lock_guard<std::mutex> guard(objMapMutex);
      if (obj->count.load(std::memory_order_acquire) == 0) {
        auto iter = objMap.find(obj->name);
        if (iter != objMap.end()) {
          objMap.erase(iter);
        }
      }
    }
  }

该方案在引用计数降至零时，在获得锁之后再次检查，这种方法咋一看没有问题，但是在逻辑上有\
漏洞。想象一下这样的场景：线程0执行 ``ObjectPut``，在得到 ``ret==1`` 时，该线程被调度出\
去，线程1调用 ``ObjectGet`` 获取 ``obj``, 获取之后然后调用 ``ObjectPut``, 此时它会\
得到 ``ret==1`` 然后销毁 ``obj``。之后线程0恢复执行，然后获取锁，测试 ``obj->count``\
但是此时 ``obj`` 已经在线程0中被销毁，因此导致未定义的行为。

-----------------------------------

与 ``std::shared_ptr`` 比较
============================
c++标准库 ``#include <memory>`` 提供了 ``shared_ptr<T>``， ``shared_ptr`` 本质上也\
是使用引用计数实现的。但是相对于上面的方案多了一层间接，这个在某些情况可能导致不低的性能\
损耗而且 ``shared_ptr`` 本身并不是线程安全的。

``std::shared_ptr`` 一个简单的实现
----------------------------------

.. code-block:: c++
  :linenos:
  :emphasize-lines: 111
  :caption: simple shared_ptr

  template<typename T>
  class shared_ptr {
    friend class weak_ptr<T>;
   private:
    class shared_ptr_impl {
     private:
      ~shared_ptr_impl() = default;

     public:
      T* ptr;
      std::atomic<int> strongRef;
      std::atomic<int> weakRef;

      explicit shared_ptr_impl(T* _ptr)
        : ptr(_ptr)
        , strongRef(1)
        , weakRef(1)
      {}

      void addStrongRef() {
        strongRef.fetch_add(1, std::memory_order_relaxed);
        this->addWeakRef();
      }

      bool tryAddStrongRef() {
        int count = strongRef.load(std::memory_order_relaxed);
        do {
          if (count == 0) {
            return false;
          }
        } while (strongRef.compare_and_swap(count, count + 1));
        this->addWeakRef();
        return true;
      }

      void releaseStrongRef() {
        int ret = strongRef.fetch_sub(1, std::memory_order_release);
        if (ret == 1) {
          std::atomic_thread_fence(std::memory_order_acquire);
          delete impl_->ptr;
          impl_->ptr = nullptr;
        }
        this->releaseWeakRef();
      }

      void addWeakRef() {
        weakRef.fetch_add(1, std::memory_order_relaxed);
      }

      void releaseWeakRef() {
        int ret = weakRef.fetch_sub(1, std::memory_order_release);
        if (ret == 1) {
          std::atomic_thread_fence(std::memory_order_acquire);
          delete this;
        }
      }
    };

    shared_ptr_impl *impl_;

    shared_ptr(shared_ptr_impl* impl)
      : impl_(impl)
    {}

   public:
    shared_ptr()
      : impl_(nullptr)
    {}

    explicit shared_ptr(T* p)
      : impl_(new shared_ptr_impl(p))
    {}

    shared_ptr(const shared_ptr& other)
      : impl_(other.impl_) {
      if (impl_) {
        impl_->addStrongRef();
      }
    }

    shared_ptr(shared_ptr&& rhs)
      : impl_(rhs.impl_)
    { rhs.impl_ = nullptr; }

    shared_ptr& operator = (const shared_ptr& other) {
      if (this == &other) {
        return (*this);
      }
      reset();
      impl_ = other.impl_;
      if (impl_) {
        impl_->addStrongRef();
      }
      return (*this);
    }

    shared_ptr& operator = (shared_ptr&& rhs) {
      if (this == &rhs) {
        return (*this);
      }
      reset();
      impl_ = rhs.impl_;
      rhs.impl_ = nullptr;
      return (*this);
    }

    operator bool() const {
      return impl_;
    }

    T* get() const { return impl_->ptr; }

    void reset() {
      if (impl_) {
        impl_->releaseStrongRef();
        impl_ = nullptr;
      }
    }

    ~shared_ptr() {
      if (impl_) {
        impl_->releaseStrongRef();
      }
    }
  };

  template<typename T>
  class weak_ptr {
   public:
    weak_ptr(const shared_ptr<T>& shared)
      : impl_(shared.impl_) {
      if (impl_) {
        impl_->addWeakRef();
      }
    }

    weak_ptr(const weak_ptr<T>& other)
      : impl_(other.impl_)
    { if (impl_) impl_->addWeakRef(); }

    weak_ptr<T> operator = (const weak_ptr<T>& other) {
      if (this == &other) {
        return (*this);
      }
      this->reset();
      this->impl_ = other.impl_;
      if (this->impl_) {
        this->impl_->addWeakRef();
      }
      return (*this);
    }

    ~weak_ptr() {
      if (impl_) {
        impl_->releaseWeakRef();
      }
    }

    void reset() {
      if (impl_) impl_->releaseWeakRef();
    }

    shared_ptr<T> take() {
      if (impl_ && impl_->tryAddStrongRef()) {
        return shared_ptr<T>(impl_);
      }
      return shared_ptr<T>();
    }
  };

从上面的实现可以看出，为了实现 ``weak_ptr`` 需要一个额外的引用计数。``weak_ptr`` 这种\
思想可以在其它地方得到应用。
