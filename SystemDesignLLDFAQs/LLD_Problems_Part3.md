# LLD Interview Problems - Part 3

## Problem 9: Rate Limiter

### Requirements
- Limit requests per user/API
- Multiple algorithms (Token Bucket, Sliding Window, Fixed Window)
- Thread-safe
- Configurable limits

### Code Implementation

```java
// ============ RATE LIMITER INTERFACE ============
public interface RateLimiter {
    boolean allowRequest(String clientId);
    void configure(int limit, long windowMs);
}

// ============ TOKEN BUCKET ALGORITHM ============
public class TokenBucketRateLimiter implements RateLimiter {
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();
    private int maxTokens;
    private long refillRateMs;

    public TokenBucketRateLimiter(int maxTokens, long refillRateMs) {
        this.maxTokens = maxTokens;
        this.refillRateMs = refillRateMs;
    }

    @Override
    public void configure(int limit, long windowMs) {
        this.maxTokens = limit;
        this.refillRateMs = windowMs / limit;
    }

    @Override
    public boolean allowRequest(String clientId) {
        TokenBucket bucket = buckets.computeIfAbsent(clientId,
            k -> new TokenBucket(maxTokens, refillRateMs));
        return bucket.tryConsume();
    }

    private static class TokenBucket {
        private final int maxTokens;
        private final long refillRateMs;
        private double tokens;
        private long lastRefillTime;
        private final ReentrantLock lock = new ReentrantLock();

        public TokenBucket(int maxTokens, long refillRateMs) {
            this.maxTokens = maxTokens;
            this.refillRateMs = refillRateMs;
            this.tokens = maxTokens;
            this.lastRefillTime = System.currentTimeMillis();
        }

        public boolean tryConsume() {
            lock.lock();
            try {
                refill();
                if (tokens >= 1) {
                    tokens--;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        private void refill() {
            long now = System.currentTimeMillis();
            long elapsed = now - lastRefillTime;
            double tokensToAdd = (double) elapsed / refillRateMs;
            tokens = Math.min(maxTokens, tokens + tokensToAdd);
            lastRefillTime = now;
        }
    }
}

// ============ SLIDING WINDOW LOG ALGORITHM ============
public class SlidingWindowLogRateLimiter implements RateLimiter {
    private final Map<String, Deque<Long>> requestLogs = new ConcurrentHashMap<>();
    private int maxRequests;
    private long windowSizeMs;

    public SlidingWindowLogRateLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
    }

    @Override
    public void configure(int limit, long windowMs) {
        this.maxRequests = limit;
        this.windowSizeMs = windowMs;
    }

    @Override
    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        Deque<Long> log = requestLogs.computeIfAbsent(clientId,
            k -> new ConcurrentLinkedDeque<>());

        synchronized (log) {
            // Remove expired timestamps
            while (!log.isEmpty() && now - log.peekFirst() > windowSizeMs) {
                log.pollFirst();
            }

            if (log.size() < maxRequests) {
                log.addLast(now);
                return true;
            }
            return false;
        }
    }
}

// ============ SLIDING WINDOW COUNTER ALGORITHM ============
public class SlidingWindowCounterRateLimiter implements RateLimiter {
    private final Map<String, WindowCounter> counters = new ConcurrentHashMap<>();
    private int maxRequests;
    private long windowSizeMs;

    public SlidingWindowCounterRateLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
    }

    @Override
    public void configure(int limit, long windowMs) {
        this.maxRequests = limit;
        this.windowSizeMs = windowMs;
    }

    @Override
    public boolean allowRequest(String clientId) {
        WindowCounter counter = counters.computeIfAbsent(clientId,
            k -> new WindowCounter(windowSizeMs));
        return counter.tryIncrement(maxRequests);
    }

    private static class WindowCounter {
        private final long windowSizeMs;
        private long currentWindowStart;
        private long previousWindowStart;
        private int currentCount;
        private int previousCount;
        private final ReentrantLock lock = new ReentrantLock();

        public WindowCounter(long windowSizeMs) {
            this.windowSizeMs = windowSizeMs;
            this.currentWindowStart = System.currentTimeMillis();
            this.previousWindowStart = currentWindowStart - windowSizeMs;
        }

        public boolean tryIncrement(int maxRequests) {
            lock.lock();
            try {
                long now = System.currentTimeMillis();
                slideWindow(now);

                // Calculate weighted count
                long elapsedInCurrentWindow = now - currentWindowStart;
                double weight = 1.0 - ((double) elapsedInCurrentWindow / windowSizeMs);
                double count = previousCount * weight + currentCount;

                if (count < maxRequests) {
                    currentCount++;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        private void slideWindow(long now) {
            if (now - currentWindowStart >= windowSizeMs) {
                previousCount = currentCount;
                previousWindowStart = currentWindowStart;
                currentCount = 0;
                currentWindowStart = now;
            }
        }
    }
}

// ============ FIXED WINDOW COUNTER ALGORITHM ============
public class FixedWindowRateLimiter implements RateLimiter {
    private final Map<String, AtomicInteger> counters = new ConcurrentHashMap<>();
    private final Map<String, Long> windowStarts = new ConcurrentHashMap<>();
    private int maxRequests;
    private long windowSizeMs;

    public FixedWindowRateLimiter(int maxRequests, long windowSizeMs) {
        this.maxRequests = maxRequests;
        this.windowSizeMs = windowSizeMs;
    }

    @Override
    public void configure(int limit, long windowMs) {
        this.maxRequests = limit;
        this.windowSizeMs = windowMs;
    }

    @Override
    public boolean allowRequest(String clientId) {
        long now = System.currentTimeMillis();
        long currentWindow = now / windowSizeMs;

        String key = clientId + ":" + currentWindow;

        windowStarts.computeIfAbsent(key, k -> now);
        AtomicInteger counter = counters.computeIfAbsent(key,
            k -> new AtomicInteger(0));

        if (counter.incrementAndGet() <= maxRequests) {
            return true;
        }
        counter.decrementAndGet();
        return false;
    }
}

// ============ LEAKY BUCKET ALGORITHM ============
public class LeakyBucketRateLimiter implements RateLimiter {
    private final Map<String, LeakyBucket> buckets = new ConcurrentHashMap<>();
    private int bucketSize;
    private long leakRateMs;

    public LeakyBucketRateLimiter(int bucketSize, long leakRateMs) {
        this.bucketSize = bucketSize;
        this.leakRateMs = leakRateMs;
    }

    @Override
    public void configure(int limit, long windowMs) {
        this.bucketSize = limit;
        this.leakRateMs = windowMs / limit;
    }

    @Override
    public boolean allowRequest(String clientId) {
        LeakyBucket bucket = buckets.computeIfAbsent(clientId,
            k -> new LeakyBucket(bucketSize, leakRateMs));
        return bucket.tryAdd();
    }

    private static class LeakyBucket {
        private final int capacity;
        private final long leakRateMs;
        private double water;
        private long lastLeakTime;
        private final ReentrantLock lock = new ReentrantLock();

        public LeakyBucket(int capacity, long leakRateMs) {
            this.capacity = capacity;
            this.leakRateMs = leakRateMs;
            this.water = 0;
            this.lastLeakTime = System.currentTimeMillis();
        }

        public boolean tryAdd() {
            lock.lock();
            try {
                leak();
                if (water < capacity) {
                    water++;
                    return true;
                }
                return false;
            } finally {
                lock.unlock();
            }
        }

        private void leak() {
            long now = System.currentTimeMillis();
            long elapsed = now - lastLeakTime;
            double leaked = (double) elapsed / leakRateMs;
            water = Math.max(0, water - leaked);
            lastLeakTime = now;
        }
    }
}

// ============ RATE LIMITER FACTORY ============
public class RateLimiterFactory {
    public enum Algorithm {
        TOKEN_BUCKET, SLIDING_WINDOW_LOG, SLIDING_WINDOW_COUNTER,
        FIXED_WINDOW, LEAKY_BUCKET
    }

    public static RateLimiter create(Algorithm algorithm, int maxRequests,
                                     long windowMs) {
        return switch (algorithm) {
            case TOKEN_BUCKET ->
                new TokenBucketRateLimiter(maxRequests, windowMs / maxRequests);
            case SLIDING_WINDOW_LOG ->
                new SlidingWindowLogRateLimiter(maxRequests, windowMs);
            case SLIDING_WINDOW_COUNTER ->
                new SlidingWindowCounterRateLimiter(maxRequests, windowMs);
            case FIXED_WINDOW ->
                new FixedWindowRateLimiter(maxRequests, windowMs);
            case LEAKY_BUCKET ->
                new LeakyBucketRateLimiter(maxRequests, windowMs / maxRequests);
        };
    }
}

// ============ API RATE LIMITER SERVICE ============
public class ApiRateLimiterService {
    private final Map<String, RateLimiter> apiLimiters = new ConcurrentHashMap<>();
    private final RateLimiter defaultLimiter;

    public ApiRateLimiterService(int defaultLimit, long defaultWindowMs) {
        this.defaultLimiter = RateLimiterFactory.create(
            RateLimiterFactory.Algorithm.SLIDING_WINDOW_COUNTER,
            defaultLimit, defaultWindowMs);
    }

    public void configureApi(String apiEndpoint,
                            RateLimiterFactory.Algorithm algorithm,
                            int limit, long windowMs) {
        apiLimiters.put(apiEndpoint,
            RateLimiterFactory.create(algorithm, limit, windowMs));
    }

    public boolean allowRequest(String clientId, String apiEndpoint) {
        RateLimiter limiter = apiLimiters.getOrDefault(apiEndpoint, defaultLimiter);
        String key = clientId + ":" + apiEndpoint;
        boolean allowed = limiter.allowRequest(key);

        if (!allowed) {
            System.out.println("Rate limit exceeded for " + clientId +
                             " on " + apiEndpoint);
        }
        return allowed;
    }
}

// ============ DEMO ============
public class RateLimiterDemo {
    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Token Bucket Rate Limiter ===\n");

        RateLimiter limiter = RateLimiterFactory.create(
            RateLimiterFactory.Algorithm.TOKEN_BUCKET, 5, 1000); // 5 per second

        String clientId = "user123";

        // Burst of requests
        for (int i = 1; i <= 8; i++) {
            boolean allowed = limiter.allowRequest(clientId);
            System.out.println("Request " + i + ": " + (allowed ? "ALLOWED" : "DENIED"));
        }

        System.out.println("\nWaiting 500ms...\n");
        Thread.sleep(500);

        for (int i = 9; i <= 12; i++) {
            boolean allowed = limiter.allowRequest(clientId);
            System.out.println("Request " + i + ": " + (allowed ? "ALLOWED" : "DENIED"));
        }

        System.out.println("\n=== API Rate Limiter Service ===\n");

        ApiRateLimiterService service = new ApiRateLimiterService(10, 1000);

        // Configure specific endpoint
        service.configureApi("/api/search",
            RateLimiterFactory.Algorithm.SLIDING_WINDOW_LOG, 3, 1000);

        for (int i = 1; i <= 5; i++) {
            boolean allowed = service.allowRequest("user1", "/api/search");
            System.out.println("/api/search Request " + i + ": " +
                             (allowed ? "ALLOWED" : "DENIED"));
        }
    }
}
```

---

## Problem 10: Online Shopping System

### Requirements
- Product catalog with categories
- Shopping cart
- Order management
- Payment processing
- Inventory management

### Code Implementation

```java
// ============ ENUMS ============
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED, RETURNED
}

public enum PaymentStatus {
    PENDING, COMPLETED, FAILED, REFUNDED
}

public enum PaymentMethod {
    CREDIT_CARD, DEBIT_CARD, UPI, NET_BANKING, WALLET, COD
}

// ============ PRODUCT ============
public class Product {
    private final String id;
    private final String name;
    private final String description;
    private final String category;
    private double price;
    private final String sellerId;

    public Product(String id, String name, String description,
                  String category, double price, String sellerId) {
        this.id = id;
        this.name = name;
        this.description = description;
        this.category = category;
        this.price = price;
        this.sellerId = sellerId;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getDescription() { return description; }
    public String getCategory() { return category; }
    public double getPrice() { return price; }
    public void setPrice(double price) { this.price = price; }
    public String getSellerId() { return sellerId; }
}

// ============ INVENTORY ============
public class Inventory {
    private final Map<String, Integer> stock = new ConcurrentHashMap<>();
    private final ReentrantLock lock = new ReentrantLock();

    public void addStock(String productId, int quantity) {
        stock.merge(productId, quantity, Integer::sum);
    }

    public boolean reserveStock(String productId, int quantity) {
        lock.lock();
        try {
            int available = stock.getOrDefault(productId, 0);
            if (available >= quantity) {
                stock.put(productId, available - quantity);
                return true;
            }
            return false;
        } finally {
            lock.unlock();
        }
    }

    public void releaseStock(String productId, int quantity) {
        stock.merge(productId, quantity, Integer::sum);
    }

    public int getAvailableStock(String productId) {
        return stock.getOrDefault(productId, 0);
    }
}

// ============ CART ITEM ============
public class CartItem {
    private final Product product;
    private int quantity;

    public CartItem(Product product, int quantity) {
        this.product = product;
        this.quantity = quantity;
    }

    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }

    public double getSubtotal() {
        return product.getPrice() * quantity;
    }
}

// ============ SHOPPING CART ============
public class ShoppingCart {
    private final String userId;
    private final Map<String, CartItem> items = new LinkedHashMap<>();

    public ShoppingCart(String userId) {
        this.userId = userId;
    }

    public void addItem(Product product, int quantity) {
        CartItem existing = items.get(product.getId());
        if (existing != null) {
            existing.setQuantity(existing.getQuantity() + quantity);
        } else {
            items.put(product.getId(), new CartItem(product, quantity));
        }
    }

    public void updateQuantity(String productId, int quantity) {
        CartItem item = items.get(productId);
        if (item != null) {
            if (quantity <= 0) {
                items.remove(productId);
            } else {
                item.setQuantity(quantity);
            }
        }
    }

    public void removeItem(String productId) {
        items.remove(productId);
    }

    public void clear() {
        items.clear();
    }

    public double getTotal() {
        return items.values().stream()
            .mapToDouble(CartItem::getSubtotal)
            .sum();
    }

    public List<CartItem> getItems() {
        return new ArrayList<>(items.values());
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }

    public String getUserId() { return userId; }
}

// ============ ORDER ITEM ============
public class OrderItem {
    private final String productId;
    private final String productName;
    private final int quantity;
    private final double priceAtPurchase;

    public OrderItem(Product product, int quantity) {
        this.productId = product.getId();
        this.productName = product.getName();
        this.quantity = quantity;
        this.priceAtPurchase = product.getPrice();
    }

    public String getProductId() { return productId; }
    public String getProductName() { return productName; }
    public int getQuantity() { return quantity; }
    public double getPriceAtPurchase() { return priceAtPurchase; }

    public double getSubtotal() {
        return priceAtPurchase * quantity;
    }
}

// ============ ADDRESS ============
public class Address {
    private final String street;
    private final String city;
    private final String state;
    private final String zipCode;
    private final String country;

    public Address(String street, String city, String state,
                  String zipCode, String country) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.zipCode = zipCode;
        this.country = country;
    }

    @Override
    public String toString() {
        return street + ", " + city + ", " + state + " " + zipCode + ", " + country;
    }
}

// ============ ORDER ============
public class Order {
    private final String id;
    private final String userId;
    private final List<OrderItem> items;
    private final Address shippingAddress;
    private final double totalAmount;
    private OrderStatus status;
    private final LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String trackingNumber;

    public Order(String id, String userId, List<OrderItem> items,
                Address shippingAddress, double totalAmount) {
        this.id = id;
        this.userId = userId;
        this.items = new ArrayList<>(items);
        this.shippingAddress = shippingAddress;
        this.totalAmount = totalAmount;
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
        this.updatedAt = createdAt;
    }

    public void updateStatus(OrderStatus status) {
        this.status = status;
        this.updatedAt = LocalDateTime.now();
    }

    public void setTrackingNumber(String trackingNumber) {
        this.trackingNumber = trackingNumber;
    }

    public String getId() { return id; }
    public String getUserId() { return userId; }
    public List<OrderItem> getItems() { return Collections.unmodifiableList(items); }
    public Address getShippingAddress() { return shippingAddress; }
    public double getTotalAmount() { return totalAmount; }
    public OrderStatus getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public String getTrackingNumber() { return trackingNumber; }
}

// ============ PAYMENT ============
public class Payment {
    private final String id;
    private final String orderId;
    private final double amount;
    private final PaymentMethod method;
    private PaymentStatus status;
    private final LocalDateTime createdAt;
    private String transactionId;

    public Payment(String id, String orderId, double amount, PaymentMethod method) {
        this.id = id;
        this.orderId = orderId;
        this.amount = amount;
        this.method = method;
        this.status = PaymentStatus.PENDING;
        this.createdAt = LocalDateTime.now();
    }

    public void complete(String transactionId) {
        this.transactionId = transactionId;
        this.status = PaymentStatus.COMPLETED;
    }

    public void fail() {
        this.status = PaymentStatus.FAILED;
    }

    public void refund() {
        this.status = PaymentStatus.REFUNDED;
    }

    public String getId() { return id; }
    public String getOrderId() { return orderId; }
    public double getAmount() { return amount; }
    public PaymentStatus getStatus() { return status; }
}

// ============ PAYMENT PROCESSOR ============
public interface PaymentProcessor {
    boolean process(Payment payment);
}

public class CreditCardProcessor implements PaymentProcessor {
    @Override
    public boolean process(Payment payment) {
        // Simulate payment processing
        System.out.println("Processing credit card payment: $" + payment.getAmount());
        payment.complete("CC-" + System.currentTimeMillis());
        return true;
    }
}

public class UPIProcessor implements PaymentProcessor {
    @Override
    public boolean process(Payment payment) {
        System.out.println("Processing UPI payment: $" + payment.getAmount());
        payment.complete("UPI-" + System.currentTimeMillis());
        return true;
    }
}

// ============ USER ============
public class User {
    private final String id;
    private final String name;
    private final String email;
    private final List<Address> addresses;
    private final ShoppingCart cart;

    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.addresses = new ArrayList<>();
        this.cart = new ShoppingCart(id);
    }

    public void addAddress(Address address) {
        addresses.add(address);
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public List<Address> getAddresses() { return Collections.unmodifiableList(addresses); }
    public ShoppingCart getCart() { return cart; }
}

// ============ PRODUCT CATALOG ============
public class ProductCatalog {
    private final Map<String, Product> products = new ConcurrentHashMap<>();
    private final Map<String, List<Product>> productsByCategory = new ConcurrentHashMap<>();

    public void addProduct(Product product) {
        products.put(product.getId(), product);
        productsByCategory.computeIfAbsent(product.getCategory(),
            k -> new ArrayList<>()).add(product);
    }

    public Product getProduct(String productId) {
        return products.get(productId);
    }

    public List<Product> searchByName(String query) {
        return products.values().stream()
            .filter(p -> p.getName().toLowerCase().contains(query.toLowerCase()))
            .collect(Collectors.toList());
    }

    public List<Product> getByCategory(String category) {
        return productsByCategory.getOrDefault(category, Collections.emptyList());
    }

    public List<Product> getAllProducts() {
        return new ArrayList<>(products.values());
    }
}

// ============ ORDER SERVICE ============
public class OrderService {
    private final Map<String, Order> orders = new ConcurrentHashMap<>();
    private final Map<String, List<Order>> ordersByUser = new ConcurrentHashMap<>();
    private final Inventory inventory;
    private final Map<PaymentMethod, PaymentProcessor> paymentProcessors;
    private final AtomicLong orderCounter = new AtomicLong(0);
    private final AtomicLong paymentCounter = new AtomicLong(0);

    public OrderService(Inventory inventory) {
        this.inventory = inventory;
        this.paymentProcessors = new EnumMap<>(PaymentMethod.class);
        paymentProcessors.put(PaymentMethod.CREDIT_CARD, new CreditCardProcessor());
        paymentProcessors.put(PaymentMethod.UPI, new UPIProcessor());
    }

    public Optional<Order> createOrder(User user, Address shippingAddress,
                                        PaymentMethod paymentMethod) {
        ShoppingCart cart = user.getCart();
        if (cart.isEmpty()) {
            System.out.println("Cart is empty");
            return Optional.empty();
        }

        // Reserve inventory
        for (CartItem item : cart.getItems()) {
            if (!inventory.reserveStock(item.getProduct().getId(), item.getQuantity())) {
                System.out.println("Insufficient stock for: " + item.getProduct().getName());
                // Rollback previous reservations
                rollbackReservations(cart.getItems(), item.getProduct().getId());
                return Optional.empty();
            }
        }

        // Create order
        String orderId = "ORD" + orderCounter.incrementAndGet();
        List<OrderItem> orderItems = cart.getItems().stream()
            .map(ci -> new OrderItem(ci.getProduct(), ci.getQuantity()))
            .collect(Collectors.toList());

        Order order = new Order(orderId, user.getId(), orderItems,
                               shippingAddress, cart.getTotal());

        // Process payment
        String paymentId = "PAY" + paymentCounter.incrementAndGet();
        Payment payment = new Payment(paymentId, orderId,
                                      cart.getTotal(), paymentMethod);

        PaymentProcessor processor = paymentProcessors.get(paymentMethod);
        if (processor == null || !processor.process(payment)) {
            // Release inventory
            for (CartItem item : cart.getItems()) {
                inventory.releaseStock(item.getProduct().getId(), item.getQuantity());
            }
            System.out.println("Payment failed");
            return Optional.empty();
        }

        order.updateStatus(OrderStatus.CONFIRMED);
        orders.put(orderId, order);
        ordersByUser.computeIfAbsent(user.getId(), k -> new ArrayList<>()).add(order);

        // Clear cart
        cart.clear();

        System.out.println("Order created: " + orderId);
        return Optional.of(order);
    }

    private void rollbackReservations(List<CartItem> items, String failedProductId) {
        for (CartItem item : items) {
            if (item.getProduct().getId().equals(failedProductId)) {
                break;
            }
            inventory.releaseStock(item.getProduct().getId(), item.getQuantity());
        }
    }

    public void shipOrder(String orderId, String trackingNumber) {
        Order order = orders.get(orderId);
        if (order != null && order.getStatus() == OrderStatus.CONFIRMED) {
            order.setTrackingNumber(trackingNumber);
            order.updateStatus(OrderStatus.SHIPPED);
            System.out.println("Order shipped: " + orderId);
        }
    }

    public void deliverOrder(String orderId) {
        Order order = orders.get(orderId);
        if (order != null && order.getStatus() == OrderStatus.SHIPPED) {
            order.updateStatus(OrderStatus.DELIVERED);
            System.out.println("Order delivered: " + orderId);
        }
    }

    public boolean cancelOrder(String orderId) {
        Order order = orders.get(orderId);
        if (order == null) return false;

        if (order.getStatus() == OrderStatus.PENDING ||
            order.getStatus() == OrderStatus.CONFIRMED) {

            // Release inventory
            for (OrderItem item : order.getItems()) {
                inventory.releaseStock(item.getProductId(), item.getQuantity());
            }

            order.updateStatus(OrderStatus.CANCELLED);
            System.out.println("Order cancelled: " + orderId);
            return true;
        }
        return false;
    }

    public List<Order> getOrderHistory(String userId) {
        return ordersByUser.getOrDefault(userId, Collections.emptyList());
    }

    public Order getOrder(String orderId) {
        return orders.get(orderId);
    }
}

// ============ SHOPPING SYSTEM ============
public class ShoppingSystem {
    private static ShoppingSystem instance;

    private final ProductCatalog catalog;
    private final Inventory inventory;
    private final OrderService orderService;
    private final Map<String, User> users = new ConcurrentHashMap<>();
    private final AtomicLong idCounter = new AtomicLong(0);

    private ShoppingSystem() {
        this.catalog = new ProductCatalog();
        this.inventory = new Inventory();
        this.orderService = new OrderService(inventory);
    }

    public static synchronized ShoppingSystem getInstance() {
        if (instance == null) {
            instance = new ShoppingSystem();
        }
        return instance;
    }

    public User registerUser(String name, String email) {
        String id = "U" + idCounter.incrementAndGet();
        User user = new User(id, name, email);
        users.put(id, user);
        return user;
    }

    public Product addProduct(String name, String description, String category,
                             double price, String sellerId, int stock) {
        String id = "P" + idCounter.incrementAndGet();
        Product product = new Product(id, name, description, category, price, sellerId);
        catalog.addProduct(product);
        inventory.addStock(id, stock);
        return product;
    }

    public void addToCart(String userId, String productId, int quantity) {
        User user = users.get(userId);
        Product product = catalog.getProduct(productId);

        if (user != null && product != null) {
            if (inventory.getAvailableStock(productId) >= quantity) {
                user.getCart().addItem(product, quantity);
                System.out.println("Added to cart: " + product.getName() + " x " + quantity);
            } else {
                System.out.println("Insufficient stock");
            }
        }
    }

    public Optional<Order> checkout(String userId, Address address,
                                    PaymentMethod paymentMethod) {
        User user = users.get(userId);
        if (user == null) return Optional.empty();

        return orderService.createOrder(user, address, paymentMethod);
    }

    public List<Product> searchProducts(String query) {
        return catalog.searchByName(query);
    }

    public ProductCatalog getCatalog() { return catalog; }
    public OrderService getOrderService() { return orderService; }
    public User getUser(String userId) { return users.get(userId); }
}

// ============ DEMO ============
public class OnlineShoppingDemo {
    public static void main(String[] args) {
        ShoppingSystem system = ShoppingSystem.getInstance();

        // Add products
        Product laptop = system.addProduct("MacBook Pro", "Apple Laptop",
                                          "Electronics", 1999.99, "seller1", 10);
        Product phone = system.addProduct("iPhone 15", "Apple Phone",
                                         "Electronics", 999.99, "seller1", 20);
        Product book = system.addProduct("Clean Code", "Programming Book",
                                        "Books", 49.99, "seller2", 50);

        // Register user
        User user = system.registerUser("John Doe", "john@email.com");
        user.addAddress(new Address("123 Main St", "New York", "NY", "10001", "USA"));

        // Add to cart
        system.addToCart(user.getId(), laptop.getId(), 1);
        system.addToCart(user.getId(), book.getId(), 2);

        // View cart
        System.out.println("\nCart Total: $" + user.getCart().getTotal());

        // Checkout
        Optional<Order> order = system.checkout(user.getId(),
            user.getAddresses().get(0), PaymentMethod.CREDIT_CARD);

        order.ifPresent(o -> {
            System.out.println("\nOrder Details:");
            System.out.println("Order ID: " + o.getId());
            System.out.println("Status: " + o.getStatus());
            System.out.println("Total: $" + o.getTotalAmount());
            System.out.println("Items:");
            o.getItems().forEach(item ->
                System.out.println("  - " + item.getProductName() +
                                 " x " + item.getQuantity() +
                                 " = $" + item.getSubtotal()));

            // Ship order
            system.getOrderService().shipOrder(o.getId(), "TRACK123456");
            System.out.println("Status: " + system.getOrderService()
                              .getOrder(o.getId()).getStatus());
        });
    }
}
```

---

## Problem 11: Vending Machine

### Requirements
- Multiple products with prices
- Coin/cash handling
- Product selection and dispensing
- State management
- Refund handling

### Code Implementation

```java
// ============ ENUMS ============
public enum Coin {
    PENNY(1),
    NICKEL(5),
    DIME(10),
    QUARTER(25),
    DOLLAR(100);

    private final int valueInCents;

    Coin(int valueInCents) {
        this.valueInCents = valueInCents;
    }

    public int getValue() { return valueInCents; }
}

// ============ PRODUCT ============
public class VendingProduct {
    private final String code;
    private final String name;
    private final int priceInCents;

    public VendingProduct(String code, String name, int priceInCents) {
        this.code = code;
        this.name = name;
        this.priceInCents = priceInCents;
    }

    public String getCode() { return code; }
    public String getName() { return name; }
    public int getPrice() { return priceInCents; }

    @Override
    public String toString() {
        return code + ": " + name + " ($" + String.format("%.2f", priceInCents / 100.0) + ")";
    }
}

// ============ PRODUCT SLOT ============
public class ProductSlot {
    private final VendingProduct product;
    private int quantity;

    public ProductSlot(VendingProduct product, int quantity) {
        this.product = product;
        this.quantity = quantity;
    }

    public VendingProduct getProduct() { return product; }
    public int getQuantity() { return quantity; }

    public boolean isAvailable() {
        return quantity > 0;
    }

    public void dispense() {
        if (quantity > 0) {
            quantity--;
        }
    }

    public void restock(int amount) {
        quantity += amount;
    }
}

// ============ COIN INVENTORY ============
public class CoinInventory {
    private final Map<Coin, Integer> coins = new EnumMap<>(Coin.class);

    public CoinInventory() {
        for (Coin coin : Coin.values()) {
            coins.put(coin, 0);
        }
    }

    public void add(Coin coin, int count) {
        coins.merge(coin, count, Integer::sum);
    }

    public void remove(Coin coin, int count) {
        coins.compute(coin, (k, v) -> Math.max(0, (v == null ? 0 : v) - count));
    }

    public int getCount(Coin coin) {
        return coins.getOrDefault(coin, 0);
    }

    public int getTotalValue() {
        return coins.entrySet().stream()
            .mapToInt(e -> e.getKey().getValue() * e.getValue())
            .sum();
    }

    public Map<Coin, Integer> getChange(int amountInCents) {
        Map<Coin, Integer> change = new EnumMap<>(Coin.class);
        int remaining = amountInCents;

        // Try to give change using largest coins first
        Coin[] coinOrder = {Coin.DOLLAR, Coin.QUARTER, Coin.DIME, Coin.NICKEL, Coin.PENNY};

        for (Coin coin : coinOrder) {
            if (remaining >= coin.getValue()) {
                int needed = remaining / coin.getValue();
                int available = getCount(coin);
                int toGive = Math.min(needed, available);

                if (toGive > 0) {
                    change.put(coin, toGive);
                    remaining -= toGive * coin.getValue();
                }
            }
        }

        if (remaining > 0) {
            return null; // Cannot make exact change
        }

        return change;
    }

    public void dispenseChange(Map<Coin, Integer> change) {
        change.forEach(this::remove);
    }
}

// ============ VENDING MACHINE STATE ============
public interface VendingState {
    void insertCoin(VendingMachine machine, Coin coin);
    void selectProduct(VendingMachine machine, String code);
    void dispense(VendingMachine machine);
    void cancelTransaction(VendingMachine machine);
}

public class IdleVendingState implements VendingState {
    @Override
    public void insertCoin(VendingMachine machine, Coin coin) {
        machine.addToCurrentBalance(coin.getValue());
        machine.getCoinInventory().add(coin, 1);
        machine.setState(new HasMoneyVendingState());
        System.out.println("Inserted: " + coin + ". Balance: $" +
                          String.format("%.2f", machine.getCurrentBalance() / 100.0));
    }

    @Override
    public void selectProduct(VendingMachine machine, String code) {
        System.out.println("Please insert money first.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please insert money and select a product.");
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        System.out.println("No transaction to cancel.");
    }
}

public class HasMoneyVendingState implements VendingState {
    @Override
    public void insertCoin(VendingMachine machine, Coin coin) {
        machine.addToCurrentBalance(coin.getValue());
        machine.getCoinInventory().add(coin, 1);
        System.out.println("Inserted: " + coin + ". Balance: $" +
                          String.format("%.2f", machine.getCurrentBalance() / 100.0));
    }

    @Override
    public void selectProduct(VendingMachine machine, String code) {
        ProductSlot slot = machine.getProductSlot(code);

        if (slot == null) {
            System.out.println("Invalid product code: " + code);
            return;
        }

        if (!slot.isAvailable()) {
            System.out.println("Product sold out: " + slot.getProduct().getName());
            return;
        }

        if (machine.getCurrentBalance() < slot.getProduct().getPrice()) {
            int needed = slot.getProduct().getPrice() - machine.getCurrentBalance();
            System.out.println("Insufficient funds. Insert $" +
                              String.format("%.2f", needed / 100.0) + " more.");
            return;
        }

        machine.setSelectedProduct(slot);
        machine.setState(new DispensingVendingState());
        machine.dispense();
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first.");
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        int refund = machine.getCurrentBalance();
        Map<Coin, Integer> change = machine.getCoinInventory().getChange(refund);

        if (change != null) {
            machine.getCoinInventory().dispenseChange(change);
            System.out.println("Refunding: $" + String.format("%.2f", refund / 100.0));
            printCoins(change);
        }

        machine.resetTransaction();
        machine.setState(new IdleVendingState());
    }

    private void printCoins(Map<Coin, Integer> coins) {
        coins.forEach((coin, count) -> {
            if (count > 0) {
                System.out.println("  " + coin + " x " + count);
            }
        });
    }
}

public class DispensingVendingState implements VendingState {
    @Override
    public void insertCoin(VendingMachine machine, Coin coin) {
        System.out.println("Please wait, dispensing product...");
    }

    @Override
    public void selectProduct(VendingMachine machine, String code) {
        System.out.println("Please wait, dispensing product...");
    }

    @Override
    public void dispense(VendingMachine machine) {
        ProductSlot slot = machine.getSelectedProduct();
        VendingProduct product = slot.getProduct();

        // Dispense product
        slot.dispense();
        System.out.println("\nDispensing: " + product.getName());

        // Calculate and dispense change
        int changeAmount = machine.getCurrentBalance() - product.getPrice();
        if (changeAmount > 0) {
            Map<Coin, Integer> change = machine.getCoinInventory().getChange(changeAmount);
            if (change != null) {
                machine.getCoinInventory().dispenseChange(change);
                System.out.println("Change: $" + String.format("%.2f", changeAmount / 100.0));
                change.forEach((coin, count) -> {
                    if (count > 0) {
                        System.out.println("  " + coin + " x " + count);
                    }
                });
            }
        }

        System.out.println("Thank you for your purchase!");

        machine.resetTransaction();
        machine.setState(new IdleVendingState());
    }

    @Override
    public void cancelTransaction(VendingMachine machine) {
        System.out.println("Cannot cancel, dispensing in progress.");
    }
}

// ============ VENDING MACHINE ============
public class VendingMachine {
    private final Map<String, ProductSlot> productSlots;
    private final CoinInventory coinInventory;
    private VendingState state;
    private int currentBalance;
    private ProductSlot selectedProduct;

    public VendingMachine() {
        this.productSlots = new LinkedHashMap<>();
        this.coinInventory = new CoinInventory();
        this.state = new IdleVendingState();
        this.currentBalance = 0;

        // Initialize with some change
        coinInventory.add(Coin.QUARTER, 20);
        coinInventory.add(Coin.DIME, 20);
        coinInventory.add(Coin.NICKEL, 20);
        coinInventory.add(Coin.PENNY, 50);
    }

    public void addProduct(VendingProduct product, int quantity) {
        productSlots.put(product.getCode(), new ProductSlot(product, quantity));
    }

    public void restockProduct(String code, int quantity) {
        ProductSlot slot = productSlots.get(code);
        if (slot != null) {
            slot.restock(quantity);
            System.out.println("Restocked " + slot.getProduct().getName() +
                             ". New quantity: " + slot.getQuantity());
        }
    }

    public void insertCoin(Coin coin) {
        state.insertCoin(this, coin);
    }

    public void selectProduct(String code) {
        state.selectProduct(this, code);
    }

    public void dispense() {
        state.dispense(this);
    }

    public void cancel() {
        state.cancelTransaction(this);
    }

    public void displayProducts() {
        System.out.println("\n=== Available Products ===");
        productSlots.values().forEach(slot -> {
            String availability = slot.isAvailable() ?
                "(Qty: " + slot.getQuantity() + ")" : "(SOLD OUT)";
            System.out.println(slot.getProduct() + " " + availability);
        });
        System.out.println();
    }

    // State management
    public void setState(VendingState state) { this.state = state; }
    public void addToCurrentBalance(int amount) { this.currentBalance += amount; }
    public int getCurrentBalance() { return currentBalance; }
    public void resetTransaction() {
        this.currentBalance = 0;
        this.selectedProduct = null;
    }
    public ProductSlot getProductSlot(String code) { return productSlots.get(code); }
    public ProductSlot getSelectedProduct() { return selectedProduct; }
    public void setSelectedProduct(ProductSlot slot) { this.selectedProduct = slot; }
    public CoinInventory getCoinInventory() { return coinInventory; }
}

// ============ DEMO ============
public class VendingMachineDemo {
    public static void main(String[] args) {
        VendingMachine machine = new VendingMachine();

        // Add products
        machine.addProduct(new VendingProduct("A1", "Coca Cola", 150), 5);
        machine.addProduct(new VendingProduct("A2", "Pepsi", 150), 5);
        machine.addProduct(new VendingProduct("B1", "Chips", 100), 3);
        machine.addProduct(new VendingProduct("B2", "Chocolate Bar", 125), 4);
        machine.addProduct(new VendingProduct("C1", "Water", 100), 10);

        // Display products
        machine.displayProducts();

        // Scenario 1: Successful purchase
        System.out.println("=== Scenario 1: Buy Chips ===");
        machine.insertCoin(Coin.DOLLAR);
        machine.selectProduct("B1");

        // Scenario 2: Purchase with exact change
        System.out.println("\n=== Scenario 2: Buy Coca Cola ===");
        machine.insertCoin(Coin.DOLLAR);
        machine.insertCoin(Coin.QUARTER);
        machine.insertCoin(Coin.QUARTER);
        machine.selectProduct("A1");

        // Scenario 3: Insufficient funds
        System.out.println("\n=== Scenario 3: Insufficient Funds ===");
        machine.insertCoin(Coin.QUARTER);
        machine.selectProduct("A1"); // Needs $1.50

        // Scenario 4: Cancel transaction
        System.out.println("\n=== Scenario 4: Cancel Transaction ===");
        machine.cancel();

        // Display updated inventory
        machine.displayProducts();
    }
}
```

---

## Problem 12: Cab Booking System (Uber/Ola)

### Requirements
- Driver and rider management
- Ride booking and matching
- Fare calculation
- Real-time tracking (simplified)
- Rating system

### Code Implementation

```java
// ============ ENUMS ============
public enum RideStatus {
    REQUESTED, DRIVER_ASSIGNED, DRIVER_ARRIVED, IN_PROGRESS, COMPLETED, CANCELLED
}

public enum DriverStatus {
    AVAILABLE, BUSY, OFFLINE
}

public enum VehicleType {
    BIKE(5.0, 1),
    AUTO(8.0, 3),
    MINI(10.0, 4),
    SEDAN(12.0, 4),
    SUV(15.0, 6),
    PREMIUM(20.0, 4);

    private final double baseFarePerKm;
    private final int capacity;

    VehicleType(double baseFarePerKm, int capacity) {
        this.baseFarePerKm = baseFarePerKm;
        this.capacity = capacity;
    }

    public double getBaseFarePerKm() { return baseFarePerKm; }
    public int getCapacity() { return capacity; }
}

// ============ LOCATION ============
public class Location {
    private final double latitude;
    private final double longitude;

    public Location(double latitude, double longitude) {
        this.latitude = latitude;
        this.longitude = longitude;
    }

    public double distanceTo(Location other) {
        // Simplified distance calculation (Haversine would be used in production)
        double latDiff = Math.abs(this.latitude - other.latitude);
        double lonDiff = Math.abs(this.longitude - other.longitude);
        return Math.sqrt(latDiff * latDiff + lonDiff * lonDiff) * 111; // ~111 km per degree
    }

    public double getLatitude() { return latitude; }
    public double getLongitude() { return longitude; }

    @Override
    public String toString() {
        return String.format("(%.4f, %.4f)", latitude, longitude);
    }
}

// ============ VEHICLE ============
public class Vehicle {
    private final String id;
    private final String licensePlate;
    private final VehicleType type;
    private final String model;
    private final String color;

    public Vehicle(String id, String licensePlate, VehicleType type,
                  String model, String color) {
        this.id = id;
        this.licensePlate = licensePlate;
        this.type = type;
        this.model = model;
        this.color = color;
    }

    public String getId() { return id; }
    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
    public String getModel() { return model; }
    public String getColor() { return color; }
}

// ============ RIDER ============
public class Rider {
    private final String id;
    private final String name;
    private final String phone;
    private final String email;
    private double rating;
    private int totalRides;

    public Rider(String id, String name, String phone, String email) {
        this.id = id;
        this.name = name;
        this.phone = phone;
        this.email = email;
        this.rating = 5.0;
        this.totalRides = 0;
    }

    public void addRating(double newRating) {
        rating = ((rating * totalRides) + newRating) / (totalRides + 1);
        totalRides++;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getPhone() { return phone; }
    public double getRating() { return rating; }
}

// ============ DRIVER ============
public class Driver {
    private final String id;
    private final String name;
    private final String phone;
    private final String licenseNumber;
    private final Vehicle vehicle;
    private Location currentLocation;
    private DriverStatus status;
    private double rating;
    private int totalRides;
    private double totalEarnings;

    public Driver(String id, String name, String phone,
                 String licenseNumber, Vehicle vehicle) {
        this.id = id;
        this.name = name;
        this.phone = phone;
        this.licenseNumber = licenseNumber;
        this.vehicle = vehicle;
        this.status = DriverStatus.OFFLINE;
        this.rating = 5.0;
        this.totalRides = 0;
        this.totalEarnings = 0;
    }

    public void goOnline(Location location) {
        this.currentLocation = location;
        this.status = DriverStatus.AVAILABLE;
        System.out.println("Driver " + name + " is now online at " + location);
    }

    public void goOffline() {
        this.status = DriverStatus.OFFLINE;
        System.out.println("Driver " + name + " is now offline");
    }

    public void updateLocation(Location location) {
        this.currentLocation = location;
    }

    public void addRating(double newRating) {
        rating = ((rating * totalRides) + newRating) / (totalRides + 1);
        totalRides++;
    }

    public void addEarnings(double amount) {
        totalEarnings += amount;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getPhone() { return phone; }
    public Vehicle getVehicle() { return vehicle; }
    public Location getCurrentLocation() { return currentLocation; }
    public DriverStatus getStatus() { return status; }
    public void setStatus(DriverStatus status) { this.status = status; }
    public double getRating() { return rating; }
    public double getTotalEarnings() { return totalEarnings; }
}

// ============ RIDE ============
public class Ride {
    private final String id;
    private final Rider rider;
    private Driver driver;
    private final Location pickupLocation;
    private final Location dropLocation;
    private final VehicleType vehicleType;
    private RideStatus status;
    private final LocalDateTime requestTime;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
    private double fare;
    private double distance;
    private Integer riderRating;
    private Integer driverRating;

    public Ride(String id, Rider rider, Location pickup, Location drop,
               VehicleType vehicleType) {
        this.id = id;
        this.rider = rider;
        this.pickupLocation = pickup;
        this.dropLocation = drop;
        this.vehicleType = vehicleType;
        this.status = RideStatus.REQUESTED;
        this.requestTime = LocalDateTime.now();
        this.distance = pickup.distanceTo(drop);
    }

    public void assignDriver(Driver driver) {
        this.driver = driver;
        this.status = RideStatus.DRIVER_ASSIGNED;
        driver.setStatus(DriverStatus.BUSY);
    }

    public void driverArrived() {
        this.status = RideStatus.DRIVER_ARRIVED;
    }

    public void startRide() {
        this.status = RideStatus.IN_PROGRESS;
        this.startTime = LocalDateTime.now();
    }

    public void completeRide(double actualDistance) {
        this.status = RideStatus.COMPLETED;
        this.endTime = LocalDateTime.now();
        this.distance = actualDistance;
        this.driver.setStatus(DriverStatus.AVAILABLE);
    }

    public void cancel() {
        this.status = RideStatus.CANCELLED;
        if (driver != null) {
            driver.setStatus(DriverStatus.AVAILABLE);
        }
    }

    public void setFare(double fare) { this.fare = fare; }
    public void setRiderRating(int rating) { this.riderRating = rating; }
    public void setDriverRating(int rating) { this.driverRating = rating; }

    public String getId() { return id; }
    public Rider getRider() { return rider; }
    public Driver getDriver() { return driver; }
    public Location getPickupLocation() { return pickupLocation; }
    public Location getDropLocation() { return dropLocation; }
    public VehicleType getVehicleType() { return vehicleType; }
    public RideStatus getStatus() { return status; }
    public double getFare() { return fare; }
    public double getDistance() { return distance; }
    public Integer getRiderRating() { return riderRating; }
    public Integer getDriverRating() { return driverRating; }
}

// ============ PRICING STRATEGY ============
public interface PricingStrategy {
    double calculateFare(Ride ride);
}

public class StandardPricingStrategy implements PricingStrategy {
    private static final double BASE_FARE = 30.0;
    private static final double PER_MINUTE_RATE = 1.0;

    @Override
    public double calculateFare(Ride ride) {
        double distanceFare = ride.getDistance() *
                             ride.getVehicleType().getBaseFarePerKm();

        // Simplified time calculation
        double timeFare = (ride.getDistance() / 30) * 60 * PER_MINUTE_RATE;

        return BASE_FARE + distanceFare + timeFare;
    }
}

public class SurgePricingStrategy implements PricingStrategy {
    private final double surgeMultiplier;
    private final PricingStrategy basePricing;

    public SurgePricingStrategy(double surgeMultiplier) {
        this.surgeMultiplier = surgeMultiplier;
        this.basePricing = new StandardPricingStrategy();
    }

    @Override
    public double calculateFare(Ride ride) {
        return basePricing.calculateFare(ride) * surgeMultiplier;
    }
}

// ============ DRIVER MATCHING STRATEGY ============
public interface DriverMatchingStrategy {
    Optional<Driver> findDriver(List<Driver> availableDrivers,
                                Location pickupLocation, VehicleType vehicleType);
}

public class NearestDriverStrategy implements DriverMatchingStrategy {
    @Override
    public Optional<Driver> findDriver(List<Driver> availableDrivers,
                                        Location pickupLocation,
                                        VehicleType vehicleType) {
        return availableDrivers.stream()
            .filter(d -> d.getStatus() == DriverStatus.AVAILABLE)
            .filter(d -> d.getVehicle().getType() == vehicleType)
            .min(Comparator.comparingDouble(d ->
                d.getCurrentLocation().distanceTo(pickupLocation)));
    }
}

public class HighestRatedDriverStrategy implements DriverMatchingStrategy {
    private static final double MAX_DISTANCE_KM = 5.0;

    @Override
    public Optional<Driver> findDriver(List<Driver> availableDrivers,
                                        Location pickupLocation,
                                        VehicleType vehicleType) {
        return availableDrivers.stream()
            .filter(d -> d.getStatus() == DriverStatus.AVAILABLE)
            .filter(d -> d.getVehicle().getType() == vehicleType)
            .filter(d -> d.getCurrentLocation().distanceTo(pickupLocation) <= MAX_DISTANCE_KM)
            .max(Comparator.comparingDouble(Driver::getRating));
    }
}

// ============ RIDE SERVICE ============
public class RideService {
    private final Map<String, Ride> rides = new ConcurrentHashMap<>();
    private final Map<String, List<Ride>> ridesByRider = new ConcurrentHashMap<>();
    private final Map<String, List<Ride>> ridesByDriver = new ConcurrentHashMap<>();
    private final List<Driver> drivers;
    private final DriverMatchingStrategy matchingStrategy;
    private PricingStrategy pricingStrategy;
    private final AtomicLong rideCounter = new AtomicLong(0);

    public RideService(List<Driver> drivers) {
        this.drivers = drivers;
        this.matchingStrategy = new NearestDriverStrategy();
        this.pricingStrategy = new StandardPricingStrategy();
    }

    public void setSurgePricing(double multiplier) {
        this.pricingStrategy = new SurgePricingStrategy(multiplier);
        System.out.println("Surge pricing activated: " + multiplier + "x");
    }

    public void resetPricing() {
        this.pricingStrategy = new StandardPricingStrategy();
        System.out.println("Standard pricing restored");
    }

    public Optional<Ride> requestRide(Rider rider, Location pickup,
                                       Location drop, VehicleType vehicleType) {
        System.out.println("\n" + rider.getName() + " requesting " + vehicleType +
                          " from " + pickup + " to " + drop);

        // Find available driver
        Optional<Driver> driverOpt = matchingStrategy.findDriver(
            drivers, pickup, vehicleType);

        if (driverOpt.isEmpty()) {
            System.out.println("No drivers available for " + vehicleType);
            return Optional.empty();
        }

        // Create ride
        String rideId = "RIDE" + rideCounter.incrementAndGet();
        Ride ride = new Ride(rideId, rider, pickup, drop, vehicleType);

        // Calculate estimated fare
        double fare = pricingStrategy.calculateFare(ride);
        ride.setFare(fare);

        System.out.println("Estimated fare: $" + String.format("%.2f", fare));
        System.out.println("Estimated distance: " +
                          String.format("%.1f", ride.getDistance()) + " km");

        // Assign driver
        Driver driver = driverOpt.get();
        ride.assignDriver(driver);

        System.out.println("Driver assigned: " + driver.getName() +
                          " (" + driver.getVehicle().getModel() + ", " +
                          driver.getVehicle().getLicensePlate() + ")");
        System.out.println("Driver rating: " +
                          String.format("%.1f", driver.getRating()) + " stars");

        rides.put(rideId, ride);
        ridesByRider.computeIfAbsent(rider.getId(), k -> new ArrayList<>()).add(ride);
        ridesByDriver.computeIfAbsent(driver.getId(), k -> new ArrayList<>()).add(ride);

        return Optional.of(ride);
    }

    public void driverArrived(String rideId) {
        Ride ride = rides.get(rideId);
        if (ride != null && ride.getStatus() == RideStatus.DRIVER_ASSIGNED) {
            ride.driverArrived();
            System.out.println("Driver has arrived at pickup location");
        }
    }

    public void startRide(String rideId) {
        Ride ride = rides.get(rideId);
        if (ride != null && ride.getStatus() == RideStatus.DRIVER_ARRIVED) {
            ride.startRide();
            System.out.println("Ride started");
        }
    }

    public void completeRide(String rideId, double actualDistance) {
        Ride ride = rides.get(rideId);
        if (ride != null && ride.getStatus() == RideStatus.IN_PROGRESS) {
            ride.completeRide(actualDistance);

            // Recalculate fare based on actual distance
            Ride actualRide = new Ride(rideId, ride.getRider(),
                                       ride.getPickupLocation(), ride.getDropLocation(),
                                       ride.getVehicleType());
            // Use reflection or recreate with actual distance
            double finalFare = pricingStrategy.calculateFare(ride);
            ride.setFare(finalFare);

            // Update driver earnings (80% to driver)
            ride.getDriver().addEarnings(finalFare * 0.8);

            System.out.println("Ride completed!");
            System.out.println("Final fare: $" + String.format("%.2f", finalFare));
            System.out.println("Distance traveled: " +
                              String.format("%.1f", actualDistance) + " km");
        }
    }

    public void cancelRide(String rideId) {
        Ride ride = rides.get(rideId);
        if (ride != null &&
            (ride.getStatus() == RideStatus.REQUESTED ||
             ride.getStatus() == RideStatus.DRIVER_ASSIGNED)) {
            ride.cancel();
            System.out.println("Ride cancelled");
        }
    }

    public void rateDriver(String rideId, int rating) {
        Ride ride = rides.get(rideId);
        if (ride != null && ride.getStatus() == RideStatus.COMPLETED) {
            ride.setDriverRating(rating);
            ride.getDriver().addRating(rating);
            System.out.println("Rated driver: " + rating + " stars");
        }
    }

    public void rateRider(String rideId, int rating) {
        Ride ride = rides.get(rideId);
        if (ride != null && ride.getStatus() == RideStatus.COMPLETED) {
            ride.setRiderRating(rating);
            ride.getRider().addRating(rating);
            System.out.println("Rated rider: " + rating + " stars");
        }
    }

    public List<Ride> getRideHistory(String riderId) {
        return ridesByRider.getOrDefault(riderId, Collections.emptyList());
    }
}

// ============ CAB BOOKING SYSTEM ============
public class CabBookingSystem {
    private static CabBookingSystem instance;

    private final Map<String, Rider> riders = new ConcurrentHashMap<>();
    private final Map<String, Driver> drivers = new ConcurrentHashMap<>();
    private final RideService rideService;
    private final AtomicLong idCounter = new AtomicLong(0);

    private CabBookingSystem() {
        this.rideService = new RideService(new ArrayList<>(drivers.values()));
    }

    public static synchronized CabBookingSystem getInstance() {
        if (instance == null) {
            instance = new CabBookingSystem();
        }
        return instance;
    }

    public Rider registerRider(String name, String phone, String email) {
        String id = "R" + idCounter.incrementAndGet();
        Rider rider = new Rider(id, name, phone, email);
        riders.put(id, rider);
        System.out.println("Rider registered: " + name);
        return rider;
    }

    public Driver registerDriver(String name, String phone, String licenseNumber,
                                 String licensePlate, VehicleType vehicleType,
                                 String model, String color) {
        String driverId = "D" + idCounter.incrementAndGet();
        String vehicleId = "V" + idCounter.incrementAndGet();

        Vehicle vehicle = new Vehicle(vehicleId, licensePlate, vehicleType, model, color);
        Driver driver = new Driver(driverId, name, phone, licenseNumber, vehicle);
        drivers.put(driverId, driver);

        System.out.println("Driver registered: " + name + " with " + model);
        return driver;
    }

    public Optional<Ride> bookRide(String riderId, Location pickup,
                                   Location drop, VehicleType vehicleType) {
        Rider rider = riders.get(riderId);
        if (rider == null) {
            System.out.println("Rider not found");
            return Optional.empty();
        }

        // Get available drivers
        List<Driver> availableDrivers = drivers.values().stream()
            .filter(d -> d.getStatus() == DriverStatus.AVAILABLE)
            .collect(Collectors.toList());

        RideService tempService = new RideService(availableDrivers);
        return tempService.requestRide(rider, pickup, drop, vehicleType);
    }

    public void driverGoOnline(String driverId, Location location) {
        Driver driver = drivers.get(driverId);
        if (driver != null) {
            driver.goOnline(location);
        }
    }

    public void driverGoOffline(String driverId) {
        Driver driver = drivers.get(driverId);
        if (driver != null) {
            driver.goOffline();
        }
    }

    public RideService getRideService() { return rideService; }
    public Driver getDriver(String id) { return drivers.get(id); }
    public Rider getRider(String id) { return riders.get(id); }
}

// ============ DEMO ============
public class CabBookingDemo {
    public static void main(String[] args) {
        CabBookingSystem system = CabBookingSystem.getInstance();

        // Register drivers
        Driver driver1 = system.registerDriver("John Driver", "111-111-1111",
            "DL001", "ABC-1234", VehicleType.SEDAN, "Honda City", "White");
        Driver driver2 = system.registerDriver("Jane Driver", "222-222-2222",
            "DL002", "XYZ-5678", VehicleType.SUV, "Toyota Fortuner", "Black");
        Driver driver3 = system.registerDriver("Bob Driver", "333-333-3333",
            "DL003", "PQR-9999", VehicleType.MINI, "Maruti Swift", "Red");

        // Drivers go online
        system.driverGoOnline(driver1.getId(), new Location(12.9716, 77.5946)); // Bangalore
        system.driverGoOnline(driver2.getId(), new Location(12.9800, 77.5900));
        system.driverGoOnline(driver3.getId(), new Location(12.9650, 77.6000));

        // Register rider
        Rider rider = system.registerRider("Alice Rider", "444-444-4444",
                                           "alice@email.com");

        // Book a ride
        Location pickup = new Location(12.9716, 77.5946);
        Location drop = new Location(12.9352, 77.6245);

        List<Driver> availableDrivers = List.of(driver1, driver2, driver3);
        RideService rideService = new RideService(availableDrivers);

        Optional<Ride> rideOpt = rideService.requestRide(
            rider, pickup, drop, VehicleType.SEDAN);

        rideOpt.ifPresent(ride -> {
            // Simulate ride flow
            rideService.driverArrived(ride.getId());
            rideService.startRide(ride.getId());

            // Complete ride with actual distance
            rideService.completeRide(ride.getId(), 8.5);

            // Rate each other
            rideService.rateDriver(ride.getId(), 5);
            rideService.rateRider(ride.getId(), 4);

            // Show driver earnings
            System.out.println("\nDriver earnings: $" +
                String.format("%.2f", ride.getDriver().getTotalEarnings()));
        });

        // Test surge pricing
        System.out.println("\n=== Surge Pricing Test ===");
        rideService.setSurgePricing(1.5);

        Optional<Ride> surgeRide = rideService.requestRide(
            rider, pickup, drop, VehicleType.SUV);

        surgeRide.ifPresent(ride -> {
            System.out.println("Surge fare applied!");
        });
    }
}
```

---

## Summary

All 12 LLD problems have been implemented:

| # | Problem | Key Patterns Used |
|---|---------|-------------------|
| 1 | BookMyShow | Strategy, Factory, Singleton |
| 2 | Elevator System | Strategy, State, Observer |
| 3 | Library Management | Repository, Service Layer |
| 4 | ATM Machine | State Pattern |
| 5 | Tic-Tac-Toe | Strategy Pattern |
| 6 | Chess | Template Method, Strategy |
| 7 | Hotel Booking | Repository, Service Layer |
| 8 | LRU Cache | Doubly Linked List + HashMap |
| 9 | Rate Limiter | Strategy, Factory |
| 10 | Online Shopping | Strategy, Factory, Service Layer |
| 11 | Vending Machine | State Pattern |
| 12 | Cab Booking | Strategy, Observer |

Each implementation includes:
- Proper class design following SOLID principles
- Thread safety where applicable
- Extensible architecture
- Working demo code
