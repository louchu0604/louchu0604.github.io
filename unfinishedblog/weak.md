背景：
一般情况下，`weak`修饰对象，`assign`修饰基本数据类型。
首先，`weak`和`assign`，并不会增加对象的rc
如果对象用`assign`来修饰，那么出栈后，`assign`修饰的对象所指向的内存块极有可能被回收，如果此时再访问指针，容易出现野指针的情况，app会因访问野指针而闪退。
那么为什么用`weak`来修饰不会出现这样的后果呢？因为`weak`所修饰的对象被系统回收之后，指针会被置为`nil`
# weak 源码
<!-- ### 源码中出现的数据结构
* `SideTables`

```objc
static StripedMap<SideTable>& SideTables() {
    return *reinterpret_cast<StripedMap<SideTable>*>(SideTableBuf);
}
```
* `SideTable`
```objc
struct SideTable {
    spinlock_t slock;
    RefcountMap refcnts;
    weak_table_t weak_table;

    SideTable() {
        memset(&weak_table, 0, sizeof(weak_table));
    }

    ~SideTable() {
        _objc_fatal("Do not delete SideTable.");
    }

    void lock() { slock.lock(); }
    void unlock() { slock.unlock(); }
    void forceReset() { slock.forceReset(); }

    // Address-ordered lock discipline for a pair of side tables.

    template<HaveOld, HaveNew>
    static void lockTwo(SideTable *lock1, SideTable *lock2);
    template<HaveOld, HaveNew>
    static void unlockTwo(SideTable *lock1, SideTable *lock2);
};
``` -->

<!-- * `RefcountMap`
* `weak_table_t` -->
```objc
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```
#### 先来看一下源码的注释，理解一下做法。

```
// Update a weak variable.
// If HaveOld is true, the variable has an existing value 
//   that needs to be cleaned up. This value might be nil.
// If HaveNew is true, there is a new value that needs to be 
//   assigned into the variable. This value might be nil.
// If CrashIfDeallocating is true, the process is halted if newObj is 
//   deallocating or newObj's class does not support weak references. 
//   If CrashIfDeallocating is false, nil is stored instead.
```
* 这个接口是用来更新weak变量的
* 如果HaveOld是ture，weak变量已经指向了某个对象，就先清除这个对象，然后指向新的对象，这个对象可能是nil
* 如果HaveNew是ture 有新的对象来替换，那么要将这个对象存起来。这个新的对象可能是nil
* 如果crashifdeallocation是ture，那么如果指向的对象已经被释放或者该对象不支持weak引用，这个进程会停止
* 如果crashifdeallocation是false，nil会被储存起来

### 然后祭出源码
```objc
id
objc_storeWeak(id *location, id newObj)
{
    return storeWeak<DoHaveOld, DoHaveNew, DoCrashIfDeallocating>
        (location, (objc_object *)newObj);
}

enum CrashIfDeallocating {
    DontCrashIfDeallocating = false, DoCrashIfDeallocating = true
};
template <HaveOld haveOld, HaveNew haveNew,
          CrashIfDeallocating crashIfDeallocating>
static id 
//传入的 location 指向旧的引用对象 newobj是新的引用对象
storeWeak(id *location, objc_object *newObj)
{
    
    assert(haveOld  ||  haveNew);
    
    if (!haveNew) assert(newObj == nil);

    Class previouslyInitializedClass = nil;
    id oldObj;
  
    SideTable *oldTable;
    SideTable *newTable;

    // Acquire locks for old and new values.
    // Order by lock address to prevent lock ordering problems. 
    // Retry if the old value changes underneath us.
 retry:
    if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
    if (haveNew) {
        newTable = &SideTables()[newObj];
    } else {
        newTable = nil;
    }
    // 接下来会有读写操作，加锁，防止数据竞争
    SideTable::lockTwo<haveOld, haveNew>(oldTable, newTable);
    //如果有旧值，但是location与oldObj不同，说明location已经被处理过了

    if (haveOld  &&  *location != oldObj) {
        SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
        goto retry;
    }

    // Prevent a deadlock between the weak reference machinery
    // and the +initialize machinery by ensuring that no 
    // weakly-referenced object has an un-+initialized isa.

   
    if (haveNew  &&  newObj) {
        //新引用的isa指正
        Class cls = newObj->getIsa();
         //if cls非空&&未初始化
        if (cls != previouslyInitializedClass  &&  
            !((objc_class *)cls)->isInitialized()) 
        {
            //解锁
            SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
            // 初始化
            _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

            // If this class is finished with +initialize then we're good.
            // If this class is still running +initialize on this thread 
            // (i.e. +initialize called storeWeak on an instance of itself)
            // then we may proceed but it will appear initializing and 
            // not yet initialized to the check above.
            // Instead set previouslyInitializedClass to recognize it on retry.
            //更新初始化过的cls
            previouslyInitializedClass = cls;
           
            goto retry;
        }
    }

    // Clean up old value, if any. 清除旧值
    if (haveOld) {
        weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
    }

    // Assign new value, if any. 分配新值
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.设置标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race. 只能在这里修改location
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
    //处理完成 解锁
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);

//返回newObj
    return (id)newObj;
}
```
检查是否有新值或旧值（haveOld==false&&haveNew==false的情况可能发生吗？）
```
assert(haveOld  ||  haveNew);
```
如果没有新值，判断传入的newObj
```
if (!haveNew) assert(newObj == nil);
```
创建两张表
```
SideTable *oldTable;
SideTable *newTable;
```
``` SideTables``` : 一个全局的Hash Map，用来管理所有对象的引用计数和weak指针。
* 如果有旧值，以oldObj为索引 从SideTables取出相应的结构体
* 如果有新值，以newOb为索引 取出相应的结构体
```objc
if (haveOld) {
        oldObj = *location;
        oldTable = &SideTables()[oldObj];
    } else {
        oldTable = nil;
    }
if (haveNew) {
    newTable = &SideTables()[newObj];
} else {
    newTable = nil;
}
```
判断cls是否为空以及是否初始化 
```objc
if (cls != previouslyInitializedClass  &&  
    !((objc_class *)cls)->isInitialized()) 
{
    //解锁
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
    // 初始化
    _class_initialize(_class_getNonMetaClass(cls, (id)newObj));

    // If this class is finished with +initialize then we're good.
    // If this class is still running +initialize on this thread 
    // (i.e. +initialize called storeWeak on an instance of itself)
    // then we may proceed but it will appear initializing and 
    // not yet initialized to the check above.
    // Instead set previouslyInitializedClass to recognize it on retry.
    //更新初始化过的cls
    previouslyInitializedClass = cls;
    
    goto retry;
}
```
清除旧值
```objc
if (haveOld) {
    weak_unregister_no_lock(&oldTable->weak_table, oldObj, location);
}
```
分配新值

```objc
    // Assign new value, if any. 
    if (haveNew) {
        newObj = (objc_object *)
            weak_register_no_lock(&newTable->weak_table, (id)newObj, location, 
                                  crashIfDeallocating);
        // weak_register_no_lock returns nil if weak store should be rejected

        // Set is-weakly-referenced bit in refcount table.设置标记位
        if (newObj  &&  !newObj->isTaggedPointer()) {
            newObj->setWeaklyReferenced_nolock();
        }

        // Do not set *location anywhere else. That would introduce a race. 只能在这里修改location
        *location = (id)newObj;
    }
    else {
        // No new value. The storage is not changed.
    }
```
处理完成后
```objc
  //解锁
    SideTable::unlockTwo<haveOld, haveNew>(oldTable, newTable);
  //返回newObj
    return (id)newObj;
```



# 相关细节：

### `weak_unregister_no_lock` 移除旧的指针:
objc-weak.mm line 348
```objc
//weak_table referent_id要指向的对象 *referrer_id指针
void
weak_unregister_no_lock(weak_table_t *weak_table, id referent_id, 
                        id *referrer_id)
{
    //旧的指针
    objc_object *referent = (objc_object *)referent_id;
    //指向。。的指针
    objc_object **referrer = (objc_object **)referrer_id;

    weak_entry_t *entry;
   //判断
    if (!referent) return;
// 判断weak_table是否存在对该对象的weak_entry
//weak_entry_for_referent ：返回对象的weak_entry 
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        //移除该entry中的指针
        remove_referrer(entry, referrer);
        bool empty = true;
        if (entry->out_of_line()  &&  entry->num_refs != 0) {
            empty = false;
        }
        else {
            for (size_t i = 0; i < WEAK_INLINE_COUNT; i++) {
                if (entry->inline_referrers[i]) {
                    empty = false; 
                    break;
                }
            }
        }

        if (empty) {
            weak_entry_remove(weak_table, entry);
        }
    }

    // Do not set *referrer = nil. objc_storeWeak() requires that the 
    // value not change.
}
```


 * The global weak references table. Stores object ids as keys,
 * and weak_entry_t structs as their values.

```objc
struct weak_table_t {
    weak_entry_t *weak_entries;
    size_t    num_entries;
    uintptr_t mask;
    uintptr_t max_hash_displacement;
};
```
### `weak_register_no_lock` 新的指针:
objc-weak.mm line 391
```objc
id 
weak_register_no_lock(weak_table_t *weak_table, id referent_id, 
                      id *referrer_id, bool crashIfDeallocating)
{
    objc_object *referent = (objc_object *)referent_id;
    objc_object **referrer = (objc_object **)referrer_id;

    if (!referent  ||  referent->isTaggedPointer()) return referent_id;

    // ensure that the referenced object is viable
    bool deallocating;
    if (!referent->ISA()->hasCustomRR()) {
        deallocating = referent->rootIsDeallocating();
    }
    else {
        BOOL (*allowsWeakReference)(objc_object *, SEL) = 
            (BOOL(*)(objc_object *, SEL))
            object_getMethodImplementation((id)referent, 
                                           SEL_allowsWeakReference);
        if ((IMP)allowsWeakReference == _objc_msgForward) {
            return nil;
        }
        deallocating =
            ! (*allowsWeakReference)(referent, SEL_allowsWeakReference);
    }

    if (deallocating) {
        if (crashIfDeallocating) {
            _objc_fatal("Cannot form weak reference to instance (%p) of "
                        "class %s. It is possible that this object was "
                        "over-released, or is in the process of deallocation.",
                        (void*)referent, object_getClassName((id)referent));
        } else {
            return nil;
        }
    }

    // now remember it and where it is being stored
    weak_entry_t *entry;
    // 判断weak_table是否存在对该对象的weak_entry
    if ((entry = weak_entry_for_referent(weak_table, referent))) {
        //该entry中新增指针

        append_referrer(entry, referrer);
    } 
    else {
        // 新增该对象的weak_entry
        weak_entry_t new_entry(referent, referrer);
        weak_grow_maybe(weak_table);
        weak_entry_insert(weak_table, &new_entry);
    }

    // Do not set *referrer. objc_storeWeak() requires that the 
    // value not change.

    return referent_id;
}

```
未完待续

对象释放,引用计数为0时 的调用顺序如下：
* objc_release
    * dealloc
        * _objc_rootDealloc
            * object_dispose
                * objc_destructInstance
                    * objc_clear_deallocating
                        * clearDeallocating
                            * clearDeallocating_slow
                                * weak_clear_no_lock

```objc
//referent_id 要释放的对象
void 
weak_clear_no_lock(weak_table_t *weak_table, id referent_id) 
{
    objc_object *referent = (objc_object *)referent_id;

    weak_entry_t *entry = weak_entry_for_referent(weak_table, referent);
    if (entry == nil) {
        /// XXX shouldn't happen, but does with mismatched CF/objc
        //printf("XXX no entry for clear deallocating %p\n", referent);
        return;
    }

    // zero out references
    weak_referrer_t *referrers;
    size_t count;
    
    if (entry->out_of_line()) {
        referrers = entry->referrers;
        count = TABLE_SIZE(entry);
    } 
    else {
        referrers = entry->inline_referrers;
        count = WEAK_INLINE_COUNT;
    }
    
    for (size_t i = 0; i < count; ++i) {
        objc_object **referrer = referrers[i];
        if (referrer) {
            if (*referrer == referent) {
                *referrer = nil;
            }
            else if (*referrer) {
                _objc_inform("__weak variable at %p holds %p instead of %p. "
                             "This is probably incorrect use of "
                             "objc_storeWeak() and objc_loadWeak(). "
                             "Break on objc_weak_error to debug.\n", 
                             referrer, (void*)*referrer, (void*)referent);
                objc_weak_error();
            }
        }
    }
    
    weak_entry_remove(weak_table, entry);
}
```


对象赋值给weak对象时 流程示意