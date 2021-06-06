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