# Leptjson

milo-yip的json解析器教程
[tutorial链接](https://github.com/miloyip/json-tutorial)  
非常好的json解析器教程，鉴于第八单元的答案没有给出，现在自己做完后记录一下。  
如有错误，请指出  

## 1. 元素比较

问题描述：完成 `lept_is_equal()` 里的对象比较部分。不需要考虑对象内有重复键的情况。

跟数组的比较类似，但是要同时检查key是否相同。  
由于对象里键值对是不考虑顺序顶的，且比较时不考虑重复键。故使用`lept_find_object_index()`。找到相同的key后再比较value  

~~~c
case LEPT_OBJECT:
                /* \todo */
                if (lhs->u.o.size != rhs->u.o.size)
                    return 0;
                for (i = 0; i < lhs->u.o.size; i++)
                {
                    size_t idx;
                    if ((idx = lept_find_object_index(rhs, lhs->u.o.m[i].k, lhs->u.o.m[i].klen)) == LEPT_KEY_NOT_EXIST)
                        return 0;
                    if (!lept_is_equal(&lhs->u.o.m[i].v, &rhs->u.o.m[idx].v))
                        return 0;
                }
                return 1;
~~~

## 2. array访问

问题描述：打开 `test_access_array()` 里的 `#if 0`，实现 `lept_insert_array_element()`、`lept_erase_array_element()` 和 `lept_clear_array()`。

### `lept_insert_array_element()`

这个函数的作用是在指定的index位置腾出一个位置。  
首先检查size和capacity，看是否要扩容。接着将index位置开始的元素整体向后移动一位。
最后使用`lept_init()`将index位置的元素初始化，同时size++，返回地址。

~~~c
lept_value* lept_insert_array_element(lept_value* v, size_t index) {
    assert(v != NULL && v->type == LEPT_ARRAY && index <= v->u.a.size);
    /* \todo */
    if (v->u.a.size == v->u.a.capacity)
        lept_reserve_array(v, v->u.a.capacity == 0 ? 1 : v->u.a.capacity * 2);
    memcpy(v->u.a.e + index + 1, v->u.a.e + index, (v->u.a.size - index) * sizeof(lept_value));
    lept_init(&v->u.a.e[index]);
    v->u.a.size++;
    return v->u.a.e + index;
}
~~~

### `lept_erase_array_element()`

这个函数的作用是删除从某个位置开始的，指定数目的元素
所以首先`lept_free()`释放内存，接着将数组后面剩下的元素往前移动
因为这样数组少了`count`个元素，所以最末尾的`count`元素是无效值,调用`lept_init()`初始化
最后更新size

~~~c
void lept_erase_array_element(lept_value* v, size_t index, size_t count) {
    assert(v != NULL && v->type == LEPT_ARRAY && index + count <= v->u.a.size);
    /* \todo */
    size_t i;
    for (i = index; i < index + count; i++)
        lept_free(&v->u.a.e[i]);
    memcpy(v->u.a.e + index, v->u.a.e + index + count, (v->u.a.size-index-count) * sizeof(lept_value));
    for (i = v->u.a.size - count; i < v->u.a.size; i++)
        lept_init(&v->u.a.e[i]);
    v->u.a.size -= count;
}
~~~

### `lept_clear_array()`

使用`lept_erase_array_element()`实现即可

~~~c
void lept_clear_array(lept_value* v) {
    assert(v != NULL && v->type == LEPT_ARRAY);
    lept_erase_array_element(v, 0, v->u.a.size);
}
~~~

## 3. object访问

主要是按文字的内容，参考数组的实现，完成object的访问函数

### `lept_set_object()`

释放原来的内存，初始化类型和容量，分配新的内存

~~~c
void lept_set_object(lept_value* v, size_t capacity) {
    assert(v != NULL);
    lept_free(v);
    v->type = LEPT_OBJECT;
    v->u.o.size = 0;
    v->u.o.capacity = capacity;
    v->u.o.m = capacity > 0 ? (lept_member*)malloc(capacity * sizeof(lept_member)) : NULL;
}
~~~

### `lept_get_object_capacity()`

直接跟size一样返回即可

~~~c
size_t lept_get_object_capacity(const lept_value* v) {
    assert(v != NULL && v->type == LEPT_OBJECT);
    /* \todo */
    return v->u.o.capacity;
}
~~~

### `lept_reserve_object()`

可参考数组的实现，比较`capacity`，若不够，则`realloc()`重新分配内存

~~~c
void lept_reserve_object(lept_value* v, size_t capacity) {
    assert(v != NULL && v->type == LEPT_OBJECT);
    /* \todo */
    if (v->u.o.capacity < capacity) 
    {
        v->u.o.capacity = capacity;
        v->u.o.m = (lept_member*)realloc(v->u.o.m, capacity * sizeof(lept_member));
    }
}
~~~

### `lept_shrink_object()`

异曲同工，将capacity变成和size一致,同样使用`realloc()`

~~~c
void lept_shrink_object(lept_value* v) {
    assert(v != NULL && v->type == LEPT_OBJECT);
    /* \todo */
    if (v->u.o.capacity > v->u.o.size)
    {
        v->u.o.capacity = v->u.o.size;
        v->u.o.m = (lept_member*)realloc(v->u.o.m, v->u.o.capacity * sizeof(lept_member));
    }
}
~~~

### `lept_clear_object`

遍历object,释放key对应的字符串，同时设置为空值。`lept_free()`释放值
最后更新size

~~~c
void lept_clear_object(lept_value* v) {
    assert(v != NULL && v->type == LEPT_OBJECT);
    /* \todo */
    size_t i;
    for (i = 0; i < v->u.o.size; ++i)
    {
        char* k = v->u.o.m[i].k;
        free(k);
        v->u.o.m[i].klen = 0;
        k = NULL;
        lept_free(&v->u.o.m[i].v);
    }
    v->u.o.size = 0;
}
~~~

### `lept_set_object_value()`

根据key返回对应的value，如果不存在，则新建一个键值对，key通过参数深复制达成同步  

~~~c
lept_value* lept_set_object_value(lept_value* v, const char* key, size_t klen) {
    assert(v != NULL && v->type == LEPT_OBJECT && key != NULL);
    /* \todo */
    size_t index = lept_find_object_index(v, key, klen);
    if (index != LEPT_KEY_NOT_EXIST)
        return &v->u.o.m[index].v;
    if (v->u.o.size == v->u.o.capacity)
        lept_reserve_object(v, v->u.o.capacity == 0 ? 1 : v->u.o.capacity * 2);
    size_t sz = v->u.o.size;
    v->u.o.m[sz].k = (char*)malloc(klen + 1);
    memcpy(v->u.o.m[sz].k, key, klen);
    v->u.o.m[sz].k[klen] = '\0';
    v->u.o.m[sz].klen = klen;
    lept_init(&v->u.o.m[sz].v);

    return &v->u.o.m[v->u.o.size++].v;
}
~~~

### `lept_remove_object_value()`

首先释放原先的键值对(`lept_member`)，将后面剩下的元素整体往前移动一位。
不要忘记对最后结尾的元素初始化，因为它们已经往前移动，剩下的是无效值。

~~~c
void lept_remove_object_value(lept_value* v, size_t index) {
    assert(v != NULL && v->type == LEPT_OBJECT && index < v->u.o.size);
    /* \todo */
    lept_member* rm = &v->u.o.m[index];
    free(rm->k);
    lept_free(&rm->v);
    memcpy(v->u.o.m + index, v->u.o.m + index + 1, (v->u.o.size - index - 1) * sizeof(lept_member));
    lept_member* ed = &v->u.o.m[--v->u.o.size];
    ed->k = NULL;
    ed->klen = 0;
    lept_init(&ed->v);
}
~~~

## 4. 元素复制

问题描述：完成 `lept_copy()` 里的数组和对象的复制部分。

顺利完成了之前的练习，调用对应的函数就很方便了

### array

`lept_set_array()`初始化左值，然后遍历，elementwise地复制，最后更新size

~~~c
case LEPT_ARRAY:
            /* \todo */
            sz = src->u.a.size;
            lept_set_array(dst, sz);
            for (i = 0; i < sz; ++i)
            {
                lept_copy(&dst->u.a.e[i], &src->u.a.e[i]);
                
            }
            dst->u.a.size = src->u.a.size;
            break;
~~~

### object

`lept_set_object()`初始化，跟数组类似，不过要先调用`lept_set_object_value()`生成对应的键值对，再对value复制

~~~c
case LEPT_OBJECT:
            /* \todo */
            lept_set_object(dst, src->u.o.size);
            for (i = 0; i < src->u.o.size; ++i)
            {
                lept_member* src_member = &src->u.o.m[i];
                lept_value* dst_value = lept_set_object_value(dst, src_member->k, src_member->klen);
                lept_copy(dst_value, &src_member->v);
            }
            break;
~~~

## 5. 总结
至此，运行test.c就能完全通过测试了，且没有发现memory leak。
这个教程的质量非常高，做的中间，对数据结构，内存管理，递归分析都能有更好的理解。
TDD(Test Driven Development)也是非常的有趣