---
layout:     post
title:      linux中list_head
subtitle:   linux中list_head
date:       2023-05-18
author:     xc
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - linux
    - kernel
---
# linux中list_head

```
#ifndef LIST_H
#define LIST_H

/*
 * Copied from include/linux/...
 */

#undef offsetof
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)

/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:        the pointer to the member.
 * @type:       the type of the container struct this is embedded in.
 * @member:     the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({                      \
	const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
	(type *)( (char *)__mptr - offsetof(type,member) );})

#define offsetof(type,member) (size_t)&((type*)0->member)//
巧妙之处在于将地址0强制转换为type类型的指针，从而定位到member在type结构体中的偏移。编译器认为0是一个有效的地址，从而认为0是type指针的起始地址。				
	


struct list_head {
	struct list_head *next, *prev;
};


#define LIST_HEAD_INIT(name) { &(name), &(name) }

#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)

/**
 * list_entry - get the struct for this entry
 * @ptr:	the &struct list_head pointer.
 * @type:	the type of the struct this is embedded in.
 * @member:	the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)

/**
 * list_for_each_entry	-	iterate over list of given type
 * @pos:	the type * to use as a loop cursor.
 * @head:	the head for your list.
 * @member:	the name of the list_head within the struct.
 */
#define list_for_each_entry(pos, head, member)				\
	for (pos = list_entry((head)->next, typeof(*pos), member);	\
	     &pos->member != (head); 	\
	     pos = list_entry(pos->member.next, typeof(*pos), member))

/**
 * list_for_each_entry_safe - iterate over list of given type safe against removal of list entry
 * @pos:	the type * to use as a loop cursor.
 * @n:		another type * to use as temporary storage
 * @head:	the head for your list.
 * @member:	the name of the list_head within the struct.
 */
#define list_for_each_entry_safe(pos, n, head, member)			\
	for (pos = list_entry((head)->next, typeof(*pos), member),	\
		n = list_entry(pos->member.next, typeof(*pos), member);	\
	     &pos->member != (head);					\
	     pos = n, n = list_entry(n->member.next, typeof(*n), member))

/**
 * list_empty - tests whether a list is empty
 * @head: the list to test.
 */
static inline int list_empty(const struct list_head *head)
{
	return head->next == head;
}

/*
 * Insert a new entry between two known consecutive entries.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_add(struct list_head *_new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = _new;
	_new->next = next;
	_new->prev = prev;
	prev->next = _new;
}

/**
 * list_add_tail - add a new entry
 * @new: new entry to be added
 * @head: list head to add it before
 *
 * Insert a new entry before the specified head.
 * This is useful for implementing queues.
 */
static inline void list_add_tail(struct list_head *_new, struct list_head *head)
{
	__list_add(_new, head->prev, head);
}

/*
 * Delete a list entry by making the prev/next entries
 * point to each other.
 *
 * This is only for internal list manipulation where we know
 * the prev/next entries already!
 */
static inline void __list_del(struct list_head *prev, struct list_head *next)
{
	next->prev = prev;
	prev->next = next;
}

#define LIST_POISON1  ((void *) 0x00100100)
#define LIST_POISON2  ((void *) 0x00200200)
/**
 * list_del - deletes entry from list.
 * @entry: the element to delete from the list.
 * Note: list_empty() on entry does not return true after this, the entry is
 * in an undefined state.
 */
static inline void list_del(struct list_head *entry)
{
	__list_del(entry->prev, entry->next);
	entry->next = (struct list_head*)LIST_POISON1;
	entry->prev = (struct list_head*)LIST_POISON2;
}
#endif
```

该双向链表没有数据域

## 初始化

1.动态

```
#define LIST_HEAD_INIT(name) { &(name), &(name) }

 static inline void INIT_LIST_HEAD(struct list_head *list)
 {
         list->next = list;
         list->prev = list;
 }
```
例如
```
struct list_head my_list = LIST_HEAD_INIT(my_list);
struct list_head your_list;
INIT_LIST_HEAD(&your_list);
```

2.静态

```
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```
例如
```
LIST_HEAD(rds_sock_list); 
```

## 插入
```
static inline void __list_add(struct list_head *new,
                              struct list_head *prev,
                              struct list_head *next)
{
    next->prev = new;
    new->next = next;
    new->prev = prev;
    prev->next = new;
}
/* new 插入到head之后 */
static inline void list_add(struct list_head *new, struct list_head *head)
{
    __list_add(new, head, head->next);
}
/* new 插入到head之前 */
static inline void list_add_tail(struct list_head *new, struct list_head *head)
{
    __list_add(new, head->prev, head);
}
```

## 删除

```
static inline void __list_del(struct list_head * prev, struct list_head * next)
{
    next->prev = prev;
    prev->next = next;
}
static inline void list_del(struct list_head *entry)
{
    __list_del(entry->prev, entry->next); 
}
```

## 替换

```
static inline void list_replace(struct list_head *old, struct list_head *new)
{
    new->next = old->next;
    new->next->prev = new;
    new->prev = old->prev;
    new->prev->next = new;
}

static inline void list_replace_init(struct list_head *old, 
                                    struct list_head *new)
{
    list_replace(old, new);
    INIT_LIST_HEAD(old);
}
```

## struct入口点获取

一般通过struct list_head维护链表，那么有一个指向struct list_head*的指针ptr ,如何获取它所在结构体的指针，然后访问它的成员

```
/*
 * @ptr:        &struct list_head 指针。
 * @type:       ptr所在结构体的类型。
 * @member:     list_head在type结构体中的名字。
 */
#define list_entry(ptr, type, member) \
        container_of(ptr, type, member)
#define container_of(ptr, type, member) ({                      \
        const typeof(((type *)0)->member) *__mptr = (ptr);    \
        (type *)((char *)__mptr - offsetof(type, member));})
#define offsetof(TYPE, MEMBER) ((size_t)&((TYPE *)0)->MEMBER)

/* list_first_entry中的ptr是指表头的指针。不是第一个节点的指针，如上图所示 */ 
#define list_first_entry(ptr, type, member) \
        list_entry((ptr)->next, type, member)
```

这段代码的核心就是container_of宏，它通过巧妙使用0指针，它的成员地址就是就是该成员在实际结构中的偏移，所以通过这种巧妙的转变取得了type类型的结构体的指针。有了这个指针，我们就可以访问其任何成员了。

## 遍历链表
list_for_each遍历一个链表。

```
 // @pos：struct list_head类型的指针，用于指向我们遍历的链表节点；
 // @head：我们要遍历链表的头节点；
 #define list_for_each(pos, head) \      
         for (pos = (head)->next; pos != (head); pos = pos->next)
```

__list_for_each与list_for_each完全一样，遍历一个链表，同时也不做预取。

```
 #define __list_for_each(pos, head) \
         for (pos = (head)->next; pos != (head); pos = pos->next)
```

__list_for_each_prev用于从后向前反向遍历。

```
 #define list_for_each_prev(pos, head) \
         for (pos = (head)->prev; pos != (head); pos = pos->prev)
```

list_for_each_safe安全的遍历一个链表，其机制是我们多传入一个struct list_head的指针n，用于指向pos的下一个节点，以保证我们在删除pos指向的节点时，仍能继续遍历链表的剩余节点。

```
 #define list_for_each_safe(pos, n, head) \
         for (pos = (head)->next, n = pos->next; pos != (head); \
                 pos = n, n = pos->next)
```
list_for_each_prev_safe反向遍历，安全查找。

```
 #define list_for_each_prev_safe(pos, n, head) \
         for (pos = (head)->prev, n = pos->prev; \
              pos != (head); \
              pos = n, n = pos->prev)
```
前面5项在遍历链表时返回的是struct list_head指针的地址。当我们使用struct list_head型变量将一个节点挂到一个链表时，我们不是为了仅仅操纵这个光凸凸的节点，而是将struct list_head变量放到一个结构体内，根据对链表上struct list_head的遍历来得出strcut list_head所在结构体的首地址，list_for_each_entry正是为了完成这一功能而实现。

```
 #define list_for_each_entry(pos, head, member)                          \
         for (pos = list_entry((head)->next, typeof(*pos), member);      \
              &pos->member != (head);    \
              pos = list_entry(pos->member.next, typeof(*pos), member))
```
我们将所求结构体类型的指针变量pos、链表的头head和所求结构体内struct list_head的变量名member传到list_for_each_entry之后， list_entry的第一个参数用head->next指向下一个节点，此节点的地址也就是在所属结构体内的struct list_head成员变量的地址，第二个参数用typeof(*pos)求得pos的结构体类型，第三个参数为所求结构体内struct list_head的变量名。