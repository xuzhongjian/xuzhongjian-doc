# dp合集 #

## [509]斐波那契数 ##

[leetcode_link](https://leetcode-cn.com/problems/fibonacci-number/)

```java
class Solution {

    private int[] dp;

    public int fib(int n) {
        dp = new int[n + 1];
        return subFib(n);
    }

    public int subFib(int n) {
        if (n == 0) {
            return 0;
        }
        if (n == 1) {
            dp[n] = 1;
            return 1;
        }
        int n1 = dp[n - 1] != 0
                ? dp[n - 1]
                : subFib(n - 1);

        int n2 = dp[n - 2] != 0
                ? dp[n - 2]
                : subFib(n - 2);
        dp[n] = n1 + n2;
        return dp[n];
    }
}
```

可以用递归的思路做：

```txt
3:16 PM	info
				Success:
				Runtime:10 ms, faster than 19.67% of Java online submissions.
				Memory Usage:35.1 MB, less than 72.91% of Java online submissions.
```

也可以用动态规划的思路做：

```txt
3:29 PM	info
				Success:
				Runtime:0 ms, faster than 100.00% of Java online submissions.
				Memory Usage:35.1 MB, less than 74.64% of Java online submissions.
```

这个文档主要讲述的是dp的思路，所以只贴出了动态规划的代码。并且动态规划的效率确实高！！！

## [322]零钱兑换 ##

[leetcode_link](https://leetcode-cn.com/problems/coin-change/)

```java
class Solution {

    /**
     * dp[i] 表示凑齐 i 元的金额，需要的最少的 coin 数量
     * dp[i] = min{dp[i],dp[i-coins[i]+1}
     */
    public int coinChange(int[] coins, int amount) {
        int[] dp = new int[amount + 1];
        Arrays.fill(dp, 20163844);
        dp[0] = 0;
        for (int coin : coins) {
            for (int i = coin; i <= amount; i++) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1); // --- 1
            }
        }
        return dp[amount] == 20163844 ? -1 : dp[amount];
    }
}
```

对于 `1`处的解释：

在`dp[i]`处使用到了`coins[i]` ，在`dp[j]`处也可能使用到了`coins[i]`，所以每一个硬币都可能被使用到了多次。

## [198]打家劫舍 ##

[leetcode_link](https://leetcode-cn.com/problems/house-robber/)

```java
class Solution {
	/**
     * 表示，从前 i 户人家偷到的东西的价值最大是 dp[i]
     * 转移方程：
     * dp[i] = max(dp[i - 2] + nums[i], dp[i-1])
     * 其中的第一项表示偷当前这家的收益，因为偷了当前这家，robCur = cur - 2及之前的最大收益 + 当前这家的收益
     * 其中第二项表示，不偷当前这家的收益，因为不偷当前这家，所以可以偷前一家，notRobCur =  cur - 1及之前的最大收益
     */
    private int[] dp;

    public int rob(int[] nums) {
        switch (nums.length) {
            case 1:
                return nums[0];
            case 2:
                return Math.max(nums[0], nums[1]);
            default:
                break;
        }
        dp = new int[nums.length];
        this.buildDp(nums);
        return dp[dp.length - 1];
    }

    public void buildDp(int[] nums) {
        dp[0] = nums[0];
        dp[1] = Math.max(nums[0], nums[1]);
        for (int i = 2; i < nums.length; i++) {
            int robCur = dp[i - 2] + nums[i];
            int notRobCur = dp[i - 1];
            dp[i] = Math.max(robCur, notRobCur);
        }
    }
}
```

## [53]最大子序和 ##

[leetcode_link](https://leetcode-cn.com/problems/maximum-subarray/)

```java
class Solution {
    
    /**
     * dp[i]表示：以 nums[i] 结尾的连续子数组最大的和是 dp[i]
     * 转移方程：
     * dp[i] = max(dp[i - 1] + nums[i], nums[i])
     * 第一项表示，nums[i] 要成为前面的连续子数组的下一位，拼接到原来的连续子数组之后的新的连续子数组的和是 dp[i - 1] + nums[i]
     * 第二项表示，nums[i] 是一个新的连续子数组的第一位，当然也只有一位，这个崭新的连续子数组的和就是 nums[i]
     * dp[i] 取这两项中的较大值
     */
    private int[] dp;
    private int res;
    
    public int maxSubArray(int[] nums) {
        dp = new int[nums.length];
        fullDp(nums);
        return res;
    }

    public void fullDp(int[] nums) {
        dp[0] = nums[0];
        res = dp[0];
        for (int i = 1; i < nums.length; i++) {
            dp[i] = Math.max(dp[i - 1] + nums[i], nums[i]);
            res = Math.max(dp[i], res);
        }
    }
}
```



## [300]最长递增子序列 ##

[leetcode_link](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

```java
class Solution {
    /**
     * dp[i]: 以 nums[i] 结尾的这个子序列的最长递增子序列的长度
     * dp[i] = max(dp[m]) + 1 // m < i && nums[m] < nums[i]
     */
    private int[] dp;
    private int res;

    public int lengthOfLIS(int[] nums) {
        dp = new int[nums.length];
        fillDp(nums);
        return res;
    }

    public void fillDp(int[] nums) {
        for (int i = 0; i < nums.length; i++) {
            // 需要在 index == i 之前找比 target 小的位置
            int target = nums[i];
            // 找到了 limit 之后就不需要再找比 limit 小的项了，因为已经被 cover 进 limit 里面了
            int limit = Integer.MIN_VALUE;
            int temp = 0;
            for (int m = i - 1; m >= 0; m--) {
                if (limit < nums[m] && nums[m] < target) {
                    temp = Math.max(temp, dp[m]);
                    limit = nums[m];
                }
            }
            dp[i] = temp + 1;
            res = Math.max(res, dp[i]);
        }
    }
}
```



## [10]正则表达式匹配 ##

[leetcode_link](https://leetcode-cn.com/problems/regular-expression-matching/)

```java
class Solution {

    /**
     * dp[i][j] 表示 s 的前 i 个字符与 p 中的前 j 个字符是否能够匹配
     */
    public static boolean isMatch(String s, String p) {
        boolean[][] dp = new boolean[s.length() + 1][p.length() + 1];
        dp[0][0] = true;

        for (int i = 0; i <= s.length(); i++) {
            for (int j = 1; j <= p.length(); j++) {
                if (p.charAt(j - 1) == '*') {
                    // xya*zzz -- 尝试使用字符y的位置的结果来匹配，也就是a*的重复次数为0
                    dp[i][j] = dp[i][j - 2];
                    if (!dp[i][j]) {
                        if (i == 0) {
                            continue;
                        }
                        char before = p.charAt(j - 2);
                        if (s.charAt(i - 1) == before || before == '.') {
                            dp[i][j] = dp[i - 1][j];
                        }
                    }
                } else {
                    // 普通字符
                    if (i == 0) {
                        continue;
                    }
                    // s 和 p 在对应的位置相同，或是 p 的位置为通配符 .
                    if (s.charAt(i - 1) == p.charAt(j - 1) || p.charAt(j - 1) == '.') {
                        dp[i][j] = dp[i - 1][j - 1];
                    }
                }
            }
        }

        return dp[s.length()][p.length()];
    }
}
```



## [44]通配符匹配 ##

[leetcode_link](https://leetcode-cn.com/problems/wildcard-matching/)

```java
class Solution {

    /**
     * dp[i][j] 表示 s 的前 i 个字符与 p 中的前 j 个字符是否能够匹配
     */
    public boolean isMatch(String s, String p) {
        boolean[][] dp = new boolean[s.length() + 1][p.length() + 1];
        dp[0][0] = true;

        // 因为 * 才能匹配空字符串，所以只有当 p 的前 j 个字符均为星号时，dp[0][j] 才为真
        for (int i = 1; i <= p.length(); i++) {
            if (p.charAt(i - 1) == '*') {
                dp[0][i] = true;
            } else {
                break;
            }
        }

        for (int i = 0; i <= s.length(); i++) {
            for (int j = 1; j <= p.length(); j++) {
                if (i == 0) continue;
                char c = p.charAt(j - 1);
                // c 可能是 字符、? 和 *
                if (c == '*') {
                    // * -- 第一项表示使用这个 * 那么 dp[i][j] 和 dp[i - 1][j]相同
                    //   -- 第二项表示不使用这个 * 那么 dp[i][j] 和 dp[i][j - 1]相同
                    dp[i][j] = dp[i - 1][j] || dp[i][j - 1];
                } else if (c == '?') {
                    // ? -- 通配
                    dp[i][j] = dp[i - 1][j - 1];
                } else {
                    // 字符 --  对应位置的两个字符相同
                    if (s.charAt(i - 1) == p.charAt(j - 1)) {
                        dp[i][j] = dp[i - 1][j - 1];
                    }
                }
            }
        }
        return dp[s.length()][p.length()];
    }
}
```



## [1143]最长公共子序列 ##

[leetcode_link](https://leetcode-cn.com/problems/longest-common-subsequence/)

```java
class Solution {
    /**
     * dp[i][j] 表示以字符串s1.[i]为结尾，以字符串s2.[j]为结尾的两个字符最长的公共字串的长度是 dp[i][j]
     * 转移方程：
     * dp[i][j] = Math.max(dp[i][j - 1], dp[i - 1][j])  | s2.[j] != s1.[i]
     * dp[i][j] = dp[i - 1][j - 1]                      | s2.[j] != s1.[i]
     */
    public int longestCommonSubsequence(String s1, String s2) {
        int[][] dp = new int[s1.length()][s2.length()];
        for (int i = 0; i < s1.length(); i++) {
            for (int j = 0; j < s2.length(); j++) {
                if (s1.charAt(i) == s2.charAt(j)) {
                    dp[i][j] = ((i < 1 || j < 1) ? 0 : dp[i - 1][j - 1]) + 1;
                } else {
                    dp[i][j] = Math.max(j < 1 ? 0 : dp[i][j - 1], i < 1 ? 0 : dp[i - 1][j]);
                }
            }
        }
        return dp[s1.length() - 1][s2.length() - 1];
    }
}
```