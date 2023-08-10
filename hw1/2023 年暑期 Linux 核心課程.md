## [2023 年暑期 Linux 核心課程](https://hackmd.io/@sysprog/linux2023-summer/https%3A%2F%2Fhackmd.io%2F%40sysprog%2FSJxzu7zo3)第 1 次作業

### [測驗$\alpha$](https://hackmd.io/@sysprog/linux2023-summer-quiz0#%E6%B8%AC%E9%A9%97-alpha)

可以參考[playground](https://leetcode.com/playground/CX6gJ7Qj)能show出 S-tree

AAAA = st_replace_right(del, least)  
BBBB = st_update(root, least) **正確答案應該是 st_update(&least, least)**  
CCCC = st_replace_left(del, most)  
DDDD = st_update(root, most) **正確答案應該是 st_update(&most, most)**  
EEEE = st_update(root, parent)  
FFFF = st_left(n)  
GGGG = st_right(n)  
 

#### 延伸問題:

1. 解釋上述程式碼運作原理
    
    `st_insert`:  
    每次插入新節點時都會traversal tree接著插入到正確位置，並且更新`(st_update)`結構。  
    其中`node->hint`是關鍵，判斷是否需要`rotate`，重複更新至`root`。  
    **但是有BUG需要修改!!!**  
    `st_remove`:  
    分成兩步驟，1.先找出右子樹最小值或左子樹最大值，接著做`st_replace`。  
    ~這地方我答案寫錯，因為我沒把樹印出來，印出來後發現其實根本沒有平衡XD，修改後才變平衡樹(但是作業表單已經交了，應該是對的吧)~  
    2.更新`(st_update)` `node->hint`判斷是否需要rotate，重複更新至`root`。  
    
2. 指出上述程式碼可改進之處，特別跟 AVL tree 和 red-black tree 相比，並予以實作
    * 總共修改兩個方向  
    * `st_update`及`st_rotate_right/left`  
    
    根據[FB討論](https://www.facebook.com/groups/system.software2023/permalink/709439087674915/)  
    此樹worst case的插入需要時間`O(n)`，而由於每次insert皆須更新樹高，類型偏向AVLtree，因此朝AVL tree修改，更新st_update的方式如下:  
    [GitHub](https://github.com/weiweichi/2023-summer-Jserv-Linux)
    ```c
    static inline void st_update(struct st_node **root, struct st_node *n)
    {
    if (!n)
        return;

    int b = st_balance(n);
    int prev_hint = n->hint;
    // dont need below code
    // struct st_node *p = st_parent(n); 

    if (b < -1) {
        /* leaning to the right */
        // modified here
        if (st_left(st_right(n)) && st_balance(st_right(n)) > 0)
            st_rotate_left(st_right(n));
        // modified above
        if (n == *root)
            *root = st_right(n);
        st_rotate_right(n);
    }

    else if (b > 1) {
        /* leaning to the left */
        // modified here
        if (st_right(st_left(n)) && st_balance(st_left(n)) < 0)
            st_rotate_right(st_left(n));
        // modified above
        if (n == *root)
            *root = st_left(n);
        st_rotate_left(n);
    }

    n->hint = st_max_hint(n);
    if (n->hint == 0 || n->hint != prev_hint)
        st_update(root, st_parent(n));  // modified here, update the parent
    }
    ```
    修改完之後又發現:  
    由於樹是依據`hint(height)`來判斷是否需要更新，倘若`hint`尚未更新，樹將有機會rotate錯誤!!  
    (可以在[playground](https://leetcode.com/playground/CX6gJ7Qj)中測試，line 81~84 103~106請自行註解掉)  
    因此在`st_rotate`同步`hint`的操作。  
    ```c
    static inline void st_rotate_left(struct st_node *n)
    {
        struct st_node *l = st_left(n), *p = st_parent(n);
        // modified hint
        short _t;
        _t = n->hint; n->hint = l->hint;
        l->hint = _t; 

        st_parent(l) = st_parent(n);
        st_left(n) = st_right(l);
        st_parent(n) = l;
        st_right(l) = n;

        if (p && st_left(p) == n)
            st_left(p) = l;
        else if (p)
            st_right(p) = l;

        if (st_left(n))
            st_lparent(n) = n;
    }
    static inline void st_rotate_right(struct st_node *n)
    {
        struct st_node *r = st_right(n), *p = st_parent(n);
        // modified hint
        short _t;
        _t = n->hint; n->hint = r->hint;
        r->hint = _t;

        st_parent(r) = st_parent(n);
        st_right(n) = st_left(r);
        st_parent(n) = r;
        st_left(r) = n;

        if (p && st_left(p) == n)
            st_left(p) = r;
        else if (p)
            st_right(p) = r;

        if (st_right(n))
            st_rparent(n) = n;
    }
    ```
    
    
3. 設計效能評比程式，探討上述程式碼和 red-black tree 效能落差


---

### [測驗$\beta$](https://hackmd.io/@sysprog/linux2023-summer-quiz0#%E6%B8%AC%E9%A9%97-beta)

MMMM = (sz + mask) & ~mask

#### 延伸問題:
1. 說明上述程式碼的運作原理

    雖然不加```if```程式還是能正常執行，但是除法(/)及乘法(*)運算耗時龐大，
    因此希望捨棄乘除法，利用 bit manipulation算出相同結果。
    
    **最終目標**: ```result```要是```alignment```的倍數且大於等於`sz`。
    
    `sz + mask`沒甚麼好改的，目的就是為了*無條件進位*(假如需要進位)，  
    而原本的除法及乘法是為了*捨棄餘數*。  
    **現在目標:捨棄餘數**  
    因為```alignment```已經是2的k冪(= 1<<k)->只會有一個1-bit，  
    ```mask(=aligment - 1)```剛好會是 0b0...001...1，  
    而利用bit的性質`&`之後剛好能捨棄餘數又保留倍數的值。  
    ```c
                                   k
    ex: alignment         = 0b00000100000 ->2的k幕
        mask              = 0b00000011111 
        ~mask             = 0b11111100000
        sz                = 0b01100111000
        sz+mask           = 0b01111010111 ->進位
        (sz+mask)&~mask   = 0b01111000000 ->捨棄餘數
      = result            = 0b01101000000
    ```
    
2. 在 Linux 核心原始程式碼找出類似 align_up 的程式碼，並舉例說明其用法  

    [include/linux/align.h](https://github.com/torvalds/linux/blob/master/include/linux/align.h)  
    ```c
    /* SPDX-License-Identifier: GPL-2.0 */
    #ifndef _LINUX_ALIGN_H
    #define _LINUX_ALIGN_H

    #include <linux/const.h>

    /* @a is a power of 2 value */
    #define ALIGN(x, a)		__ALIGN_KERNEL((x), (a))
    #define ALIGN_DOWN(x, a)	__ALIGN_KERNEL((x) - ((a) - 1), (a))
    #define __ALIGN_MASK(x, mask)	__ALIGN_KERNEL_MASK((x), (mask))
    #define PTR_ALIGN(p, a)		((typeof(p))ALIGN((unsigned long)(p), (a)))
    #define PTR_ALIGN_DOWN(p, a)	((typeof(p))ALIGN_DOWN((unsigned long)(p), (a)))
    #define IS_ALIGNED(x, a)		(((x) & ((typeof(x))(a) - 1)) == 0)

    #endif	/* _LINUX_ALIGN_H */
    ```
    
    [tools/include/linux/mm.h](https://github.com/torvalds/linux/blob/f0ab9f34e59e0c01a1c31142e0b336245367fd86/tools/include/linux/mm.h)  
    ```c
    /* SPDX-License-Identifier: GPL-2.0 */
    #ifndef _TOOLS_LINUX_MM_H
    #define _TOOLS_LINUX_MM_H

    #include <linux/mmzone.h>
    #include <uapi/linux/const.h>

    #define PAGE_SHIFT		12
    #define PAGE_SIZE		(_AC(1, UL) << PAGE_SHIFT)
    #define PAGE_MASK		(~(PAGE_SIZE - 1))

    #define PHYS_ADDR_MAX	(~(phys_addr_t)0)

    #define __ALIGN_KERNEL(x, a)		__ALIGN_KERNEL_MASK(x, (typeof(x))(a) - 1)
    #define __ALIGN_KERNEL_MASK(x, mask)	(((x) + (mask)) & ~(mask))
    #define ALIGN(x, a)			__ALIGN_KERNEL((x), (a))
    #define ALIGN_DOWN(x, a)		__ALIGN_KERNEL((x) - ((a) - 1), (a))

    #define PAGE_ALIGN(addr) ALIGN(addr, PAGE_SIZE)

    #define __va(x) ((void *)((unsigned long)(x)))
    #define __pa(x) ((unsigned long)(x))

    #define pfn_to_page(pfn) ((void *)((pfn) * PAGE_SIZE))

    #define phys_to_virt phys_to_virt
    static inline void *phys_to_virt(unsigned long address)
    {
        return __va(address);
    }

    void reserve_bootmem_region(phys_addr_t start, phys_addr_t end);

    static inline void totalram_pages_inc(void)
    {
    }

    static inline void totalram_pages_add(long count)
    {
    }

    #endif
    ```

:::warning
文字訊息不要用「螢幕截圖」來展現！
:notes: jserv
:::


---

### [測驗$\gamma$]

HHHH = pthread_cond_wait(&qs->cond_st, &qs->mtx_st)
JJJJ = pthread_cond_signal(&qs2->cond_st)

#### 延伸問題:
1. 解釋上述程式碼運作原理
    
2. 以 [Thread Sanitizer](https://github.com/google/sanitizers/wiki/ThreadSanitizerCppManual) 找出上述程式碼的 data race 並著手修正
    
3. 研讀 [專題: lib/sort.c](https://hackmd.io/@sysprog/B146OVeHn)，提出上述程式碼效能改進之規劃並予以實作
    