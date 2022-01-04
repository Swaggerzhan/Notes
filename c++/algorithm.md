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

### 88. 合并两个有序数组

思路: 从大到小合并，比较nums1和nums2中最大的数，放到数组尾部即可。

```C++
class Solution {
public:
     void merge(vector<int>& nums1, int m, vector<int>& nums2, int n) {
        int index = m + n - 1;
        int left = m - 1;
        int right = n - 1;
        while ( left >= 0 || right >= 0 ){
            if (left < 0){
                nums1[index] = nums2[right];
                right --;
                index --;
                continue;
            }
            if ( right < 0){
                nums1[index] = nums1[left];
                left --;
                index --;
                continue;
            }
            if (nums1[left] > nums2[right]) {
                nums1[index] = nums1[left];
                index --;
                left --;
            }else{
                nums1[index] = nums2[right];
                index --;
                right --;
            }
        }
    }
};
```

### 415. 字符串相加

编写思路: 直接反转两个string，之后主循环中判断两个字符串是否还有数据，没有数据只直接+'0'，这样编写的代码比较清晰。

```C++
class Solution {
public:
    string addStrings(string num1, string num2) {
       std::reverse(num1.begin(), num1.end());
        std::reverse(num2.begin(), num2.end());
        string res;
        int index = 0;
        bool flag = false;
        int tmp = 0;
        while ( index < num1.length() || index < num2.length()){
            tmp += index < num1.length() ? num1[index] : '0';
            tmp += index < num2.length() ? num2[index] : '0';
            tmp += flag;
            tmp -= '0';
            tmp -= '0';
            flag = tmp >= 10;
            res += (tmp % 10 + '0');
            tmp = 0;
            index ++;
        }
        if ( flag )
            res += '1';
        std::reverse(res.begin(), res.end());
        return res;
    }
};
```


### 232. 用栈实现队列

思路: 两个栈，一个作为输入，一个做输出，push的时候只放到输入栈(反过来了)，pop时候检查输出栈是否还有数据，没有的话从另外一个栈拿过来(再次反过来)。


### 215. 数组中第k大的数字

思路: 基本是快速排序的思路，当排序完后的 __[lt, rt)__ 中包含k即可直接返回，如果没有找到就往 __[l, lt-1]__ 或者 __[rt, r]__ 中继续寻找。时间复杂度O(n)

```C++
int findKlarget(vector<int> nums, int l, int r, int k){
	if (l > r)
		return -1; // 没找到直接返回-1
	// 以下是快排
	int rand_ = l + ( rand() % (r - l + 1));
	swap(nums[l], nums[rand_]);
	int piv = nums[l];
	int lt = l;
	int rt = r + 1;
	int index = l + 1;
	while ( index < rt ){
		if ( piv < nums[index] ){
			swap( nums[index], nums[lt + 1]);
			lt ++;
			index ++;
		}else if ( piv > nums[index] ){
			swap( nums[index], nums[rt - 1]);
			rt --;
		}else{
			index ++;
		}
	}
	swap(nums[l], nums[lt]);
	// 下面进行区间判断，成功直接返回即可
	if (k >=lt && k < rt){
		return nums[lt];
	}else if ( k < lt ){
		return findKlarget(nums, l, lt-1, k);
	}else {
		return findKlarget(nums, rt, r, k);
	}
}
```

这个算法返回的是以K为index的数，如果需要寻找第k个大小的话需要进行-1操作。

### 141. 环形链表

思路: 快慢指针，不怕死的循环就可以了，以块指针和块指针的下一个节点为判断条件，发现为空直接返回false即可，每次慢指针走一次，块指针走两次，块指针每走一次判断一次。O(N)

暴力法: 当然可以直接走的过程存到set中去，复杂度O(N logN)

```C++
bool hasCycle(ListNode* head){
    if (!head)
        return false;
    ListNode* pre = head; // slow pointer
    ListNode* nxt = head->next; // fast pointer
    while (nxt && nxt->next){
        if (pre == nxt)
            return true;
        pre = pre->next;
        nxt = nxt->next;
        if (pre == nxt)
            return true;
        nxt = nxt->next;
    }
    return false;
}
```


### 3. 无重复字符串的最长子串

思路: 滑动窗口，lt保存后节点，index在前，用一个set保存，每次检测index下位置的字符是否在set中，有就一只删除，直到set中不包含index处的字符即可，每次下set的大小，只需要最大即可

注: 每次字符都需要insert进入set中！

```C++
class Solution {
public:
    int lengthOfLongestSubstring(string s) {
        set<char> set_;
        int lt = 0;
        int len = 0;
        for (int i=0; i<s.length(); i++){
            while ( set_.find( s[i] ) != set_.end() ){ //  找到了
                set_.erase( s[lt] ); // 删除
                lt ++; // 左窗口跟进
            }
            set_.insert( s[i] );
            len = len > set_.size() ? len : set_.size();
        }
        return len;
    }
};
```


### 226. 反转二叉树

思路: 交换左右孩子节点，之后递归下去交换即可，遇到空即return空。

```C++
class Solution {
public:
    TreeNode* invertTree(TreeNode* root) {
        if (!root)
            return nullptr;
        ListNode* tmp = root->left;
        root->left = root->right;
        root->right = tmp;
        invertTree(root->left);
        invertTree(root->right);
        return root;
    }
};
```

### 236. 二叉树的最近公共祖先

思路: 如果节点为空，直接返回空，如果发现root节点就是q或者p，那就直接返回。没有返回就选择left子树以及right子树，对于寻找的left和right子树中如果都为空，那么也返回空，如果有q那就返回q，如果有p就返回p，如果都有，那就返回root即可。
```C++
class Solution {
public:
    TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
        if (!root)
            return nullptr;
        if (root == p)
            return p;
        if (root == q)
            return q;
        TreeNode* left = lowestCommonAncestor(root->left, p, q);
        TreeNode* right = lowestCommonAncestor(root->right, p, q);
        if (!left && !right)
            return nullptr;
        if (left && right)
            return root;
        return left ? left : right;
    }
};
```

### 101. 对称二叉树

思路: root之下的left节点和right节点是完全镜像的，之后递归形式的都是如此，有一个规律即: left->left和right->right相同，left->right和right->left相同。

```C++
class Solution {
public:
    bool dfs(TreeNode* left, TreeNode* right){
        /* 如果节点都为空就返回true */
        if (!left && !right)
            return true;
        /* 其中一个为空，一个不为空则返回false */
        if (!left || !right)
            return false;
        /* 判断节点是否相同 */
        if ( left->val != right->val )
            return false;
        /* 递归下去查询 */
        return dfs(left->left, right->right) &&
            dfs(left->right, right->left);    
    }
    bool isSymmetric(TreeNode* root) {
        if (!root)
            return true;
        return dfs(root->lfet, root->right);  
    }
};
```

### 剑指 Offer 03. 数组中重复的数字

思路: 由于题目描述中表示长度为n的数字中的数都在0~n-1之间，所以我们只需要将(数字)其放到所对应的索引处一一比对就能测出是否有重复的数字了。

```C++
class Solution {
public:
    int findRepeatNumber(vector<int>& nums) {
        for (int i=0; i<nums.size(); i++){
            while ( nums[i] != i ){
                if ( nums[nums[i]] == nums[i] )
                    return nums[i];
                swap(nums[nums[i]], nums[i]);
            }
        }
        return -1;
    }
};
```