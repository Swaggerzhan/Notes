# LeetCode

### 912. 快速排序

```C++
class Solution {
public:
    void sort(std::vector<int>&target, int l, int r){
        if (l >= r)
            return;
        int rand_ = l + (rand() % ( r - l + 1));
        std::swap(target[l], target[rand_]);
        int lt = l;
        int rt = r + 1;
        int index = l + 1;
        int piv = target[l];
        while (index < rt){ // 小于piv情况
            if (target[index] < piv){
                std::swap(target[index], target[lt + 1]);
                index ++;
                lt ++;
            }else if (target[index] > piv){ // 大于piv情况
                std::swap(target[index], target[rt - 1]);
                rt --;
            }else{ //和piv相等情况
                index ++;
            }
        }
        std::swap(target[l], target[lt]);
        sort(target, l, lt - 1);
        sort(target, rt, r);
    }
    std::vector<int> sortArray(std::vector<int>& nums) {
        sort(nums, 0, nums.size()-1);
        return nums;
    }
};
```

### 21. 合并两个有序链表

思路: 首选在l1链表和l2链表都还有节点时，循环判断下一个节点是l1小或是l2上的节点小，将其加入到新链表中，如果l1或者l2中没有节点了，直接将另外一条链表加入到新链表即可。

### 415. 字符串相加

思路: 一个主循环，只有当num1空了，num2也空了，或者没有进位了结束。

```C++
class Solution {
public:
    string addStrings(string num1, string num2) {
        int i = num1.length() - 1, j = num2.length() - 1, add = 0;
        string ans = "";
        while (i >= 0 || j >= 0 || add != 0) {
            int x = i >= 0 ? num1[i] - '0' : 0; // num1 用完了为0
            int y = j >= 0 ? num2[j] - '0' : 0; // num2 用完了为0
            int result = x + y + add;
            ans.push_back('0' + result % 10);
            add = result / 10;
            i -= 1;
            j -= 1;
        }
        // 计算完以后的答案需要翻转过来
        reverse(ans.begin(), ans.end());
        return ans;
    }
};
```

### 470. 用rand7()实现rand10()

思路:  (rand7()-1)*7 + rand7() 是相同概率。

### 206与92. 反转链表和反转部分链表

其中206思路: 只需要定义3个游动指针，循环以cur为条件，留意nxt是否为空，返回pre即可。
其中92思路: 同样定义3个游动指针，循环以cur为条件，注意前后区间关系。

### 160. 相交链表

思路: cur1在l1链表上游走，cur2在l2链表上游走，如果发现cur1和cur2相同，返回true即可，如果l1和l2结束了，分别对换，cur1在l2上游走，cur2在l1上游走，直到结束没有找到相同的直接返回即可。

记: left和right为不同的循环条件，结束时两者必定相同，不是nullptr就是答案。

```C++
ListNode* reWrite(ListNode* headA, ListNode* headB){
    if (!headA || !headB)
        return nullptr;
    ListNode* left = headA;
    ListNode* right = headB;
    /* 找到相同节点即可返回 */
    while ( left != right ){
        left = left ? left->next : headB;
        right = right ? right->next : headA;
    }
    /* 如果找到的不是相同节点，left和right也会因为nullptr相同而停下循环 */
    return left;
}
```