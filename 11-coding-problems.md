# 11 — Coding Problems (DSA Patterns, Java Concurrency Problems)

> **Priority**: 🟡 High  
> **Estimated Study Time**: 2 days  
> Markers: 🔥 = Frequently Asked | 💎 = Differentiator

---

## Section 1: DSA Patterns

### 🔥 Q1. Two Sum — Find two numbers that add up to target.

**Pattern**: HashMap lookup

```java
public int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> map = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        if (map.containsKey(complement)) {
            return new int[]{map.get(complement), i};
        }
        map.put(nums[i], i);
    }
    throw new IllegalArgumentException("No solution");
}
// Time: O(n), Space: O(n)
```

---

### 🔥 Q2. Sliding Window — Longest substring without repeating characters.

**Pattern**: Sliding Window with HashSet/HashMap

```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastSeen = new HashMap<>();
    int maxLen = 0;
    int left = 0;
    
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastSeen.containsKey(c) && lastSeen.get(c) >= left) {
            left = lastSeen.get(c) + 1; // Shrink window
        }
        lastSeen.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// Time: O(n), Space: O(min(n, alphabet_size))
```

---

### 🔥 Q3. Two Pointers — Container with most water.

**Pattern**: Two Pointers from both ends

```java
public int maxArea(int[] height) {
    int left = 0, right = height.length - 1;
    int maxWater = 0;
    
    while (left < right) {
        int width = right - left;
        int h = Math.min(height[left], height[right]);
        maxWater = Math.max(maxWater, width * h);
        
        // Move the shorter side inward
        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }
    return maxWater;
}
// Time: O(n), Space: O(1)
```

---

### 🔥 Q4. Binary Search — Search in rotated sorted array.

**Pattern**: Modified Binary Search

```java
public int search(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (nums[mid] == target) return mid;
        
        // Left half is sorted
        if (nums[left] <= nums[mid]) {
            if (target >= nums[left] && target < nums[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        // Right half is sorted
        else {
            if (target > nums[mid] && target <= nums[right]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    return -1;
}
// Time: O(log n), Space: O(1)
```

---

### 🔥 Q5. Linked List — Detect cycle and find cycle start.

**Pattern**: Floyd's Tortoise and Hare

```java
public ListNode detectCycle(ListNode head) {
    ListNode slow = head, fast = head;
    
    // Phase 1: Detect cycle
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) break; // Cycle detected
    }
    
    if (fast == null || fast.next == null) return null; // No cycle
    
    // Phase 2: Find cycle start
    slow = head;
    while (slow != fast) {
        slow = slow.next;
        fast = fast.next;
    }
    return slow; // Cycle start node
}
// Time: O(n), Space: O(1)
```

---

### 🔥 Q6. Tree — Lowest Common Ancestor of a Binary Tree.

**Pattern**: Recursive DFS

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) return root;
    
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    
    if (left != null && right != null) return root; // p and q on different sides
    return left != null ? left : right;
}
// Time: O(n), Space: O(h) where h = height
```

---

### 🔥 Q7. Dynamic Programming — Coin Change (minimum coins).

**Pattern**: Bottom-up DP

```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1); // Initialize with impossible value
    dp[0] = 0;
    
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return dp[amount] > amount ? -1 : dp[amount];
}
// Time: O(amount * coins.length), Space: O(amount)
```

---

### 🔥 Q8. Graph — Number of Islands (BFS/DFS).

**Pattern**: Grid DFS/BFS

```java
public int numIslands(char[][] grid) {
    int count = 0;
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == '1') {
                count++;
                dfs(grid, i, j); // Sink the island
            }
        }
    }
    return count;
}

private void dfs(char[][] grid, int i, int j) {
    if (i < 0 || i >= grid.length || j < 0 || j >= grid[0].length || grid[i][j] == '0') {
        return;
    }
    grid[i][j] = '0'; // Mark visited
    dfs(grid, i + 1, j);
    dfs(grid, i - 1, j);
    dfs(grid, i, j + 1);
    dfs(grid, i, j - 1);
}
// Time: O(m*n), Space: O(m*n) worst case recursion
```

---

### 💎 Q9. Merge Intervals.

**Pattern**: Sort + Merge

```java
public int[][] merge(int[][] intervals) {
    Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
    List<int[]> merged = new ArrayList<>();
    
    for (int[] interval : intervals) {
        if (merged.isEmpty() || merged.get(merged.size() - 1)[1] < interval[0]) {
            merged.add(interval); // No overlap
        } else {
            merged.get(merged.size() - 1)[1] = 
                Math.max(merged.get(merged.size() - 1)[1], interval[1]); // Merge
        }
    }
    return merged.toArray(new int[0][]);
}
// Time: O(n log n), Space: O(n)
```

---

### 💎 Q10. Top K Frequent Elements.

**Pattern**: HashMap + Min-Heap (or Bucket Sort)

```java
// Approach 1: Min-Heap — O(n log k)
public int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(freq::get));
    
    for (int num : freq.keySet()) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();
    }
    
    return minHeap.stream().mapToInt(Integer::intValue).toArray();
}

// Approach 2: Bucket Sort — O(n)
public int[] topKFrequentBucket(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    
    @SuppressWarnings("unchecked")
    List<Integer>[] buckets = new List[nums.length + 1];
    for (var entry : freq.entrySet()) {
        int f = entry.getValue();
        if (buckets[f] == null) buckets[f] = new ArrayList<>();
        buckets[f].add(entry.getKey());
    }
    
    List<Integer> result = new ArrayList<>();
    for (int i = buckets.length - 1; i >= 0 && result.size() < k; i--) {
        if (buckets[i] != null) result.addAll(buckets[i]);
    }
    return result.stream().mapToInt(Integer::intValue).toArray();
}
```

---

## Section 2: Java Concurrency Problems

### 🔥 Q11. Implement a thread-safe LRU Cache.

```java
public class ConcurrentLRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head; // Dummy head (most recent)
    private final Node<K, V> tail; // Dummy tail (least recent)
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    
    static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        
        Node(K key, V value) {
            this.key = key;
            this.value = value;
        }
    }
    
    public ConcurrentLRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>();
        this.head = new Node<>(null, null);
        this.tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }
    
    public V get(K key) {
        lock.readLock().lock();
        try {
            Node<K, V> node = map.get(key);
            if (node == null) return null;
            // Need write lock to move to front
            lock.readLock().unlock();
            lock.writeLock().lock();
            try {
                moveToFront(node);
                lock.readLock().lock(); // Downgrade
            } finally {
                lock.writeLock().unlock();
            }
            return node.value;
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public void put(K key, V value) {
        lock.writeLock().lock();
        try {
            Node<K, V> existing = map.get(key);
            if (existing != null) {
                existing.value = value;
                moveToFront(existing);
            } else {
                Node<K, V> newNode = new Node<>(key, value);
                map.put(key, newNode);
                addToFront(newNode);
                
                if (map.size() > capacity) {
                    Node<K, V> lru = tail.prev;
                    removeNode(lru);
                    map.remove(lru.key);
                }
            }
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    private void moveToFront(Node<K, V> node) {
        removeNode(node);
        addToFront(node);
    }
    
    private void addToFront(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
    
    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
// Time: O(1) for get and put, Space: O(capacity)
```

---

### 🔥 Q12. Implement Producer-Consumer using BlockingQueue.

```java
public class PaymentProcessingPipeline {
    
    private final BlockingQueue<PaymentRequest> queue;
    private final ExecutorService producerPool;
    private final ExecutorService consumerPool;
    private volatile boolean running = true;
    
    public PaymentProcessingPipeline(int queueCapacity, int numProducers, int numConsumers) {
        this.queue = new ArrayBlockingQueue<>(queueCapacity);
        this.producerPool = Executors.newFixedThreadPool(numProducers);
        this.consumerPool = Executors.newFixedThreadPool(numConsumers);
    }
    
    // Producer: Accepts payment requests
    public void submitPayment(PaymentRequest request) {
        producerPool.submit(() -> {
            try {
                boolean added = queue.offer(request, 5, TimeUnit.SECONDS);
                if (!added) {
                    log.warn("Queue full, rejecting payment: {}", request.getTxnId());
                    // Handle backpressure
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
    
    // Consumer: Processes payment requests
    public void startConsumers() {
        for (int i = 0; i < 3; i++) {
            consumerPool.submit(() -> {
                while (running) {
                    try {
                        PaymentRequest request = queue.poll(1, TimeUnit.SECONDS);
                        if (request != null) {
                            processPayment(request);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }
    
    private void processPayment(PaymentRequest request) {
        log.info("Processing: {}", request.getTxnId());
        // ... payment logic
    }
    
    public void shutdown() {
        running = false;
        producerPool.shutdown();
        consumerPool.shutdown();
    }
}
```

---

### 🔥 Q13. Implement a Rate Limiter (Token Bucket).

```java
public class TokenBucketRateLimiter {
    private final int maxTokens;
    private final int refillRate; // tokens per second
    private double currentTokens;
    private long lastRefillTimestamp;
    private final Object lock = new Object();
    
    public TokenBucketRateLimiter(int maxTokens, int refillRate) {
        this.maxTokens = maxTokens;
        this.refillRate = refillRate;
        this.currentTokens = maxTokens;
        this.lastRefillTimestamp = System.nanoTime();
    }
    
    public boolean tryAcquire() {
        synchronized (lock) {
            refill();
            if (currentTokens >= 1) {
                currentTokens -= 1;
                return true; // Allowed
            }
            return false; // Rate limited
        }
    }
    
    private void refill() {
        long now = System.nanoTime();
        double elapsed = (now - lastRefillTimestamp) / 1_000_000_000.0; // seconds
        double tokensToAdd = elapsed * refillRate;
        currentTokens = Math.min(maxTokens, currentTokens + tokensToAdd);
        lastRefillTimestamp = now;
    }
}

// Usage
TokenBucketRateLimiter limiter = new TokenBucketRateLimiter(100, 10); // 100 max, 10/sec refill
if (limiter.tryAcquire()) {
    processRequest();
} else {
    throw new RateLimitExceededException("Too many requests");
}
```

---

### 💎 Q14. Implement a Thread-Safe Singleton (multiple approaches).

```java
// Approach 1: Enum Singleton (BEST — recommended by Joshua Bloch)
public enum PaymentGateway {
    INSTANCE;
    
    public PaymentResult charge(PaymentRequest request) {
        // ... implementation
        return new PaymentResult("TXN-001", "SUCCESS");
    }
}
// Usage: PaymentGateway.INSTANCE.charge(request);
// Thread-safe, serialization-safe, reflection-safe

// Approach 2: Double-Checked Locking
public class PaymentGateway {
    private static volatile PaymentGateway instance; // volatile is CRITICAL
    
    private PaymentGateway() {}
    
    public static PaymentGateway getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (PaymentGateway.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new PaymentGateway();
                }
            }
        }
        return instance;
    }
}

// Approach 3: Bill Pugh Singleton (Lazy, thread-safe via class loading)
public class PaymentGateway {
    private PaymentGateway() {}
    
    private static class Holder {
        private static final PaymentGateway INSTANCE = new PaymentGateway();
        // Loaded only when Holder class is accessed
    }
    
    public static PaymentGateway getInstance() {
        return Holder.INSTANCE;
    }
}
```

---

### 💎 Q15. Print numbers 1-100 using 3 threads (T1 prints 1,4,7..., T2 prints 2,5,8..., T3 prints 3,6,9...).

```java
public class ThreeThreadPrinter {
    private final int maxNum;
    private final AtomicInteger counter = new AtomicInteger(1);
    
    public ThreeThreadPrinter(int maxNum) {
        this.maxNum = maxNum;
    }
    
    public void print(int threadId) {
        // threadId: 0, 1, or 2
        while (true) {
            int current = counter.get();
            if (current > maxNum) break;
            
            if (current % 3 == threadId) {
                System.out.println("Thread-" + threadId + ": " + current);
                counter.incrementAndGet();
            } else {
                Thread.yield(); // Let other threads run
            }
        }
    }
    
    public static void main(String[] args) {
        ThreeThreadPrinter printer = new ThreeThreadPrinter(100);
        
        // Adjust: T1 prints 1,4,7 → remainder 1 when %3
        Thread t1 = new Thread(() -> printer.print(1), "T1"); // prints 1,4,7,10...
        Thread t2 = new Thread(() -> printer.print(2), "T2"); // prints 2,5,8,11...
        Thread t3 = new Thread(() -> printer.print(0), "T3"); // prints 3,6,9,12...
        
        t1.start();
        t2.start();
        t3.start();
    }
}

// Alternative: Using wait/notify for better efficiency
public class ThreeThreadPrinterWaitNotify {
    private int current = 1;
    private final int maxNum;
    private final Object lock = new Object();
    
    public ThreeThreadPrinterWaitNotify(int maxNum) {
        this.maxNum = maxNum;
    }
    
    public void print(int threadId, int remainder) {
        synchronized (lock) {
            while (current <= maxNum) {
                if (current % 3 == remainder) {
                    System.out.println("Thread-" + threadId + ": " + current);
                    current++;
                    lock.notifyAll();
                } else {
                    try {
                        lock.wait();
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        return;
                    }
                }
            }
            lock.notifyAll(); // Wake up other threads to exit
        }
    }
}
```

---

## Section 3: DSA Pattern Cheat Sheet

| Pattern | When to Use | Key Problems |
|---------|------------|-------------|
| **Two Pointers** | Sorted array, pair finding | Two Sum II, 3Sum, Container With Most Water |
| **Sliding Window** | Subarray/substring with constraint | Max subarray, Longest substring, Min window |
| **Binary Search** | Sorted data, search space reduction | Rotated array, Peak element, Search range |
| **BFS/DFS** | Graph/tree traversal | Islands, Level order, Connected components |
| **Dynamic Programming** | Overlapping subproblems, optimal substructure | Coin change, LCS, Knapsack, House robber |
| **Backtracking** | Generate all combinations/permutations | N-Queens, Subsets, Permutations |
| **Greedy** | Local optimal → Global optimal | Interval scheduling, Jump game |
| **Stack/Monotonic Stack** | Next greater/smaller element | Daily temperatures, Largest rectangle |
| **Heap/Priority Queue** | Top K, Merge K sorted | Top K frequent, Merge K lists |
| **Union-Find** | Connected components, cycle detection | Number of provinces, Redundant connection |
| **Trie** | Prefix matching, autocomplete | Word search, Implement Trie |
| **Topological Sort** | Dependency ordering | Course schedule, Build order |

---

## Quick Revision Checklist

- [ ] Two Sum: HashMap for O(n) lookup
- [ ] Sliding Window: Two pointers, expand right, shrink left
- [ ] Binary Search: Handle sorted + rotated arrays
- [ ] DFS/BFS: Grid traversal, mark visited
- [ ] DP: Define state, transition, base case
- [ ] LRU Cache: HashMap + Doubly Linked List, O(1) operations
- [ ] Producer-Consumer: BlockingQueue, offer/poll with timeout
- [ ] Rate Limiter: Token bucket, refill based on elapsed time
- [ ] Singleton: Enum (best), Double-checked locking (volatile!), Bill Pugh
- [ ] Thread coordination: wait/notify, AtomicInteger, CountDownLatch
