Отлично — взял самые популярные **easy**-задачи (LeetCode) и подобрал для каждой **быстрое решение на Java** — где возможно время `O(n)` (указано). Под каждой задачей — ссылка на страницу задачи на LeetCode.  

(Источники: страницы задач LeetCode для каждой задачи.)  

---

# 10 популярных easy задач LeetCode + Java-решения (O(n) где возможно)

## 1) Two Sum. citeturn0search0  
Дано: массив `nums` и `target`. Вернуть индексы двух чисел, дающих сумму `target`.  
**Complexity:** O(n) time, O(n) extra space.

```java
// LeetCode style
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer,Integer> idx = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            int need = target - nums[i];
            if (idx.containsKey(need)) return new int[]{ idx.get(need), i };
            idx.put(nums[i], i);
        }
        throw new IllegalArgumentException("No solution");
    }
}
```
Пояснение: один проход HashMap — ищем дополняющее значение.  

---

## 2) Reverse Linked List (206). citeturn0search1  
Дано: head односвязного списка. Вернуть перевёрнутый список.  
**Complexity:** O(n) time, O(1) space.

```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int v){ val = v; }
}

class Solution {
    public ListNode reverseList(ListNode head) {
        ListNode prev = null, cur = head;
        while (cur != null) {
            ListNode next = cur.next;
            cur.next = prev;
            prev = cur;
            cur = next;
        }
        return prev;
    }
}
```
Пояснение: стандартный итеративный разворот указателей в одном проходе.

---

## 3) Best Time to Buy and Sell Stock (121). citeturn0search2  
Дано: массив цен `prices`, можно купить и продать один раз. Найти макс. прибыль.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int maxProfit(int[] prices) {
        int minPrice = Integer.MAX_VALUE;
        int best = 0;
        for (int p : prices) {
            if (p < minPrice) minPrice = p;
            else best = Math.max(best, p - minPrice);
        }
        return best;
    }
}
```
Пояснение: поддерживаем минимальную цену слева и текущую лучшую прибыль.

---

## 4) Valid Parentheses (20). citeturn0search3  
Дано: строка со скобками. Проверить корректность закрытия.  
**Complexity:** O(n) time, O(n) space worst-case.

```java
class Solution {
    public boolean isValid(String s) {
        Deque<Character> st = new ArrayDeque<>();
        for (char c : s.toCharArray()) {
            if (c == '(') st.push(')');
            else if (c == '[') st.push(']');
            else if (c == '{') st.push('}');
            else {
                if (st.isEmpty() || st.pop() != c) return false;
            }
        }
        return st.isEmpty();
    }
}
```
Пояснение: пушим ожидаемую закрывающую скобку, при встрече проверяем стек.

---

## 5) Maximum Subarray (53). citeturn1search0  
Дано: найти максимальную сумму непрерывного подмассива.  
**Complexity:** O(n) time, O(1) space (Kadane’s algorithm).

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int maxEnding = nums[0], maxSoFar = nums[0];
        for (int i = 1; i < nums.length; i++) {
            maxEnding = Math.max(nums[i], maxEnding + nums[i]);
            maxSoFar = Math.max(maxSoFar, maxEnding);
        }
        return maxSoFar;
    }
}
```
Пояснение: на каждом шаге либо начинаем новый подмассив, либо расширяем текущий.

---

## 6) Move Zeroes (283). citeturn1search1  
Дано: сдвинуть все нули в конец, сохраняя порядок остальных, in-place.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public void moveZeroes(int[] nums) {
        int j = 0; // позиция для следующего ненулевого
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] != 0) {
                nums[j++] = nums[i];
            }
        }
        while (j < nums.length) nums[j++] = 0;
    }
}
```
Пояснение: двухуказательный подход — сначала сжимаем все ненулевые в начало, затем заполняем нулями.

---

## 7) Contains Duplicate (217). citeturn1search2  
Дано: вернуть `true`, если в массиве есть дубликаты.  
**Complexity:** O(n) time average, O(n) space.

```java
class Solution {
    public boolean containsDuplicate(int[] nums) {
        Set<Integer> s = new HashSet<>();
        for (int n : nums) {
            if (!s.add(n)) return true;
        }
        return false;
    }
}
```
Пояснение: HashSet.add возвращает false если элемент уже есть — ранний выход.

---

## 8) Single Number (136). citeturn1search3  
Дано: все элементы встречаются дважды, кроме одного — найти его.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int singleNumber(int[] nums) {
        int res = 0;
        for (int n : nums) res ^= n;
        return res;
    }
}
```
Пояснение: XOR — парные числа аннулируются, остаётся уникальный.

---

## 9) Merge Two Sorted Lists (21). citeturn2search0  
Дано: два отсортированных списка — слить в один отсортированный, переиспользуя узлы.  
**Complexity:** O(n+m) time, O(1) extra space (реконнект узлов).

```java
class ListNode {
    int val;
    ListNode next;
    ListNode(int v){ val = v; }
}

class Solution {
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        ListNode dummy = new ListNode(0), tail = dummy;
        while (l1 != null && l2 != null) {
            if (l1.val < l2.val) { tail.next = l1; l1 = l1.next; }
            else { tail.next = l2; l2 = l2.next; }
            tail = tail.next;
        }
        tail.next = (l1 != null) ? l1 : l2;
        return dummy.next;
    }
}
```
Пояснение: классический merge двух отсортированных последовательностей.

---

## 10) Majority Element (169). citeturn2search1  
Дано: элемент, встречающийся более n/2 раз (гарантирован). Найти его.  
**Complexity:** O(n) time, O(1) space (Boyer-Moore).

```java
class Solution {
    public int majorityElement(int[] nums) {
        int candidate = 0, count = 0;
        for (int n : nums) {
            if (count == 0) candidate = n;
            count += (n == candidate) ? 1 : -1;
        }
        return candidate;
    }
}
```
Пояснение: голосование — меньшинства аннулируют, мажоритарный элемент выживет.

---

# Следующие 10 популярных задач уровня *Easy* (LeetCode) — Java-решения + краткие пояснения

Ниже — 10 популярных задач уровня *Easy* (следующие по популярности после базовых). Для каждой: краткое условие, Java-решение (часто оптимальное O(n), где применимо) и небольшое пояснение. Ссылки на страницы задач LeetCode указаны рядом с заголовками.

---

## 1) Valid Anagram (LeetCode 242). citeturn0search0
**Условие:** Даны две строки `s` и `t`. Вернуть `true`, если `t` — анаграмма `s`.  
**Complexity:** O(n) time, O(1) extra space (алфавит ограничен).

```java
class Solution {
    public boolean isAnagram(String s, String t) {
        if (s.length() != t.length()) return false;
        int[] cnt = new int[26];
        for (int i = 0; i < s.length(); i++) {
            cnt[s.charAt(i) - 'a']++;
            cnt[t.charAt(i) - 'a']--;
        }
        for (int c : cnt) if (c != 0) return false;
        return true;
    }
}
```
**Пояснение:** Одним проходом считаем плюсы для `s` и минусы для `t`. Если все нули — строки состоят из одинаковых букв.

---

## 2) Climbing Stairs (LeetCode 70). citeturn0search1
**Условие:** Сколько способов добраться до `n`-го шага, делая шаги на 1 или 2.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int climbStairs(int n) {
        if (n <= 2) return n;
        int a = 1, b = 2;
        for (int i = 3; i <= n; i++) {
            int c = a + b;
            a = b;
            b = c;
        }
        return b;
    }
}
```
**Пояснение:** Это числовая последовательность Фибоначчи; динамическое программирование с двумя переменными.

---

## 3) Valid Palindrome (LeetCode 125). citeturn0search2
**Условие:** Игнорируя небуквенно-цифровые символы и регистр, строка — палиндром?  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public boolean isPalindrome(String s) {
        int i = 0, j = s.length() - 1;
        while (i < j) {
            while (i < j && !Character.isLetterOrDigit(s.charAt(i))) i++;
            while (i < j && !Character.isLetterOrDigit(s.charAt(j))) j--;
            if (Character.toLowerCase(s.charAt(i)) != Character.toLowerCase(s.charAt(j))) return false;
            i++; j--;
        }
        return true;
    }
}
```
**Пояснение:** Два указателя, пропускаем недопустимые символы, сравниваем в нижнем регистре.

---

## 4) Reverse String (LeetCode 344). citeturn0search3
**Условие:** Дана строка в виде массива символов — развернуть in-place.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public void reverseString(char[] s) {
        int i = 0, j = s.length - 1;
        while (i < j) {
            char tmp = s[i];
            s[i++] = s[j];
            s[j--] = tmp;
        }
    }
}
```
**Пояснение:** Классический двухуказательный swap.

---

## 5) Plus One (LeetCode 66). citeturn1search0
**Условие:** Большое целое число представлено массивом цифр; прибавить 1.  
**Complexity:** O(n) time, O(1) extra space (in-place, возможен новый массив при переполнении).

```java
class Solution {
    public int[] plusOne(int[] digits) {
        for (int i = digits.length - 1; i >= 0; i--) {
            if (digits[i] < 9) {
                digits[i]++;
                return digits;
            }
            digits[i] = 0;
        }
        int[] res = new int[digits.length + 1];
        res[0] = 1;
        return res;
    }
}
```
**Пояснение:** Идём справа налево, обрабатываем перенос; только в случае всех 9 нужен новый массив.

---

## 6) Pascal's Triangle (LeetCode 118). citeturn1search1
**Условие:** Сгенерировать первые `numRows` строк треугольника Паскаля.  
**Complexity:** O(numRows^2) time, O(numRows^2) space (результат имеет такой размер).

```java
class Solution {
    public List<List<Integer>> generate(int numRows) {
        List<List<Integer>> ans = new ArrayList<>();
        for (int i = 0; i < numRows; i++) {
            List<Integer> row = new ArrayList<>(i+1);
            for (int j = 0; j <= i; j++) {
                if (j == 0 || j == i) row.add(1);
                else row.add(ans.get(i-1).get(j-1) + ans.get(i-1).get(j));
            }
            ans.add(row);
        }
        return ans;
    }
}
```
**Пояснение:** Построение строки на основе предыдущей — сумма соседних элементов.

---

## 7) Symmetric Tree (LeetCode 101). citeturn1search2
**Условие:** Проверить, является ли бинарное дерево зеркальным относительно центра.  
**Complexity:** O(n) time, O(n) space (рекурсивный стек или очередь).

```java
class TreeNode { int val; TreeNode left, right; TreeNode(int v){val=v;} }

class Solution {
    public boolean isSymmetric(TreeNode root) {
        if (root == null) return true;
        return check(root.left, root.right);
    }
    private boolean check(TreeNode a, TreeNode b) {
        if (a == null && b == null) return true;
        if (a == null || b == null) return false;
        if (a.val != b.val) return false;
        return check(a.left, b.right) && check(a.right, b.left);
    }
}
```
**Пояснение:** Рекурсивное сравнение левого поддерева с зеркалом правого.

---

## 8) Binary Tree Inorder Traversal (LeetCode 94). citeturn1search3
**Условие:** Вернуть inorder-последовательность (лево, корень, право).  
**Complexity:** O(n) time, O(n) space (результат + стек для итеративного решения).

```java
class TreeNode { int val; TreeNode left, right; TreeNode(int v){val=v;} }

class Solution {
    public List<Integer> inorderTraversal(TreeNode root) {
        List<Integer> res = new ArrayList<>();
        Deque<TreeNode> stack = new ArrayDeque<>();
        TreeNode cur = root;
        while (cur != null || !stack.isEmpty()) {
            while (cur != null) { stack.push(cur); cur = cur.left; }
            cur = stack.pop();
            res.add(cur.val);
            cur = cur.right;
        }
        return res;
    }
}
```
**Пояснение:** Итеративный DFS с явным стеком — классический паттерн.

---

## 9) Intersection of Two Arrays II (LeetCode 350). citeturn2search0
**Условие:** Вернуть пересечение двух массивов, учитывая количество вхождений.  
**Complexity:** O(m + n) time average, O(min(m,n)) extra space using хеш-таблицу.

```java
class Solution {
    public int[] intersect(int[] nums1, int[] nums2) {
        if (nums1.length > nums2.length) return intersect(nums2, nums1);
        Map<Integer, Integer> cnt = new HashMap<>();
        for (int x : nums1) cnt.put(x, cnt.getOrDefault(x, 0) + 1);
        List<Integer> res = new ArrayList<>();
        for (int x : nums2) {
            int c = cnt.getOrDefault(x, 0);
            if (c > 0) { res.add(x); cnt.put(x, c - 1); }
        }
        // convert list to array
        int[] a = new int[res.size()];
        for (int i = 0; i < res.size(); i++) a[i] = res.get(i);
        return a;
    }
}
```
**Пояснение:** Считаем частоты в меньшем массиве, затем пробегаем второй и уменьшаем счётчик при совпадении.

---

## 10) Min Stack (LeetCode 155). citeturn2search1
**Условие:** Реализовать стек с операциями `push`, `pop`, `top`, `getMin` — все за O(1).  
**Complexity:** O(1) for each operation, O(n) space.

```java
class MinStack {
    private Deque<Integer> stack = new ArrayDeque<>();
    private Deque<Integer> mins = new ArrayDeque<>();
    public void push(int val) {
        stack.push(val);
        if (mins.isEmpty() || val <= mins.peek()) mins.push(val);
        else mins.push(mins.peek());
    }
    public void pop() {
        stack.pop();
        mins.pop();
    }
    public int top() { return stack.peek(); }
    public int getMin() { return mins.peek(); }
}
```
**Пояснение:** Второй стек `mins` хранит минимум на каждом уровне — при pop оба стека синхронизируются, getMin — O(1).

---
# Ещё 10 популярных задач уровня *Easy* (LeetCode) — Java-решения + краткие пояснения

Ниже — ещё 10 часто встречающихся задач уровня *Easy*. Для каждой — условие, Java-решение (эффективное по времени) и краткое пояснение.

---

## 1) Reverse Integer (LeetCode 7)
**Условие:** Дано 32-битное целое `x`. Вернуть его цифры в обратном порядке. Если при развороте выходит за пределы 32-битного int — вернуть 0.  
**Complexity:** O(log₁₀|x|) ≈ O(1) time, O(1) space.

```java
class Solution {
    public int reverse(int x) {
        int rev = 0;
        while (x != 0) {
            int pop = x % 10;
            x /= 10;
            if (rev > Integer.MAX_VALUE/10 || (rev == Integer.MAX_VALUE/10 && pop > 7)) return 0;
            if (rev < Integer.MIN_VALUE/10 || (rev == Integer.MIN_VALUE/10 && pop < -8)) return 0;
            rev = rev * 10 + pop;
        }
        return rev;
    }
}
```
**Пояснение:** Извлекаем последнюю цифру, добавляем к `rev` с проверкой на переполнение перед умножением/сложением.

---

## 2) Palindrome Number (LeetCode 9)
**Условие:** Проверить, является ли число `x` палиндромом (без преобразования в строку предпочтительнее).  
**Complexity:** O(log₁₀ x) time, O(1) space.

```java
class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0) return false;
        int reverted = 0;
        int orig = x;
        while (x != 0) {
            reverted = reverted * 10 + x % 10;
            x /= 10;
        }
        return orig == reverted;
    }
}
```
**Пояснение:** Разворачиваем число и сравниваем с оригиналом; можно оптимизировать до разворота половины цифр, но этот вариант прост и эффективен.

---

## 3) Remove Duplicates from Sorted Array (LeetCode 26)
**Условие:** Удалить дубликаты на месте в отсортированном массиве и вернуть новую длину. Порядок должен сохраниться.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int removeDuplicates(int[] nums) {
        if (nums.length == 0) return 0;
        int j = 0;
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] != nums[j]) nums[++j] = nums[i];
        }
        return j + 1;
    }
}
```
**Пояснение:** Два указателя: `j` указывает на последний уникальный элемент, `i` сканирует массив.

---

## 4) Remove Element (LeetCode 27)
**Условие:** Удалить все вхождения `val` из массива на месте и вернуть новую длину. Элементы можно менять местами.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int removeElement(int[] nums, int val) {
        int j = 0;
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] != val) nums[j++] = nums[i];
        }
        return j;
    }
}
```
**Пояснение:** Сжимаем все элементы, не равные `val`, в начало массива.

---

## 5) Implement strStr() (LeetCode 28)
**Условие:** Найти первое вхождение `needle` в `haystack`. Вернуть индекс или -1.  
**Complexity:** O(n*m) naive, но для easy-задачи обычно достаточно. Есть KMP для O(n+m).

```java
class Solution {
    public int strStr(String haystack, String needle) {
        if (needle.isEmpty()) return 0;
        int n = haystack.length(), m = needle.length();
        for (int i = 0; i + m <= n; i++) {
            if (haystack.substring(i, i + m).equals(needle)) return i;
        }
        return -1;
    }
}
```
**Пояснение:** Простой скользящий просмотр подстрок — для длинных строк в реальных задачах лучше KMP или Z-функция.

---

## 6) Binary Search (LeetCode 704)
**Условие:** Найти `target` в отсортированном массиве, вернуть индекс или -1.  
**Complexity:** O(log n) time, O(1) space.

```java
class Solution {
    public int search(int[] nums, int target) {
        int l = 0, r = nums.length - 1;
        while (l <= r) {
            int mid = l + (r - l) / 2;
            if (nums[mid] == target) return mid;
            if (nums[mid] < target) l = mid + 1;
            else r = mid - 1;
        }
        return -1;
    }
}
```
**Пояснение:** Классическая бинарная реализация с избеганием overflow при вычислении `mid`.

---

## 7) Sqrt(x) (LeetCode 69)
**Условие:** Вычислить целую часть квадратного корня числа `x`.  
**Complexity:** O(log x) time via binary search, O(1) space.

```java
class Solution {
    public int mySqrt(int x) {
        if (x < 2) return x;
        int l = 1, r = x / 2, ans = 1;
        while (l <= r) {
            int mid = l + (r - l) / 2;
            long square = 1L * mid * mid;
            if (square == x) return mid;
            if (square < x) { ans = mid; l = mid + 1; }
            else r = mid - 1;
        }
        return ans;
    }
}
```
**Пояснение:** Бинарный поиск по возможным корням; используем `long` чтобы избежать переполнения при умножении.

---

## 8) Power of Two (LeetCode 231)
**Условие:** Проверить, является ли `n` степенью двойки.  
**Complexity:** O(1) bit trick, O(1) space.

```java
class Solution {
    public boolean isPowerOfTwo(int n) {
        if (n <= 0) return false;
        return (n & (n - 1)) == 0;
    }
}
```
**Пояснение:** Для степеней двойки в бинарном представлении ровно один бит равен 1; `n & (n-1)` обнуляет младший установленный бит.

---

## 9) Two Sum II - Input array is sorted (LeetCode 167)
**Условие:** Даны отсортированный массив и target, вернуть 1-based индексы двух чисел, дающих сумму.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int[] twoSum(int[] numbers, int target) {
        int l = 0, r = numbers.length - 1;
        while (l < r) {
            int s = numbers[l] + numbers[r];
            if (s == target) return new int[]{l + 1, r + 1};
            if (s < target) l++;
            else r--;
        }
        return new int[]{-1, -1};
    }
}
```
**Пояснение:** Два указателя снаружи к центру — используем сортировку для навигации.

---

## 10) Find Pivot Index (LeetCode 724)
**Условие:** Найти индекс `i` такого, что сумма слева от `i` равна сумме справа. Вернуть любой такой индекс или -1.  
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int pivotIndex(int[] nums) {
        int total = 0;
        for (int x : nums) total += x;
        int left = 0;
        for (int i = 0; i < nums.length; i++) {
            if (left == total - left - nums[i]) return i;
            left += nums[i];
        }
        return -1;
    }
}
```
**Пояснение:** Вычисляем общую сумму, затем на лету поддерживаем сумму слева и сравниваем с ожидаемой правой суммой.

---

Если хочешь, могу объединить этот файл с предыдущими в один большой cheat-sheet (markdown или PDF) и дать одну ссылку для скачивания.

