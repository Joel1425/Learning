# LLD Interview Problems - Part 1

## Problem 1: BookMyShow (Movie Ticket Booking System)

### Requirements
- Multiple theaters with multiple screens
- Movies with showtimes
- Seat selection and booking
- Concurrent booking handling
- Payment processing

### Code Implementation

```java
// ============ ENUMS ============
public enum SeatType {
    REGULAR(100),
    PREMIUM(150),
    VIP(250);

    private final double basePrice;

    SeatType(double basePrice) {
        this.basePrice = basePrice;
    }

    public double getBasePrice() { return basePrice; }
}

public enum SeatStatus {
    AVAILABLE, BLOCKED, BOOKED
}

public enum BookingStatus {
    PENDING, CONFIRMED, CANCELLED, EXPIRED
}

// ============ SEAT ============
public class Seat {
    private final String id;
    private final int row;
    private final int column;
    private final SeatType type;
    private SeatStatus status;

    public Seat(String id, int row, int column, SeatType type) {
        this.id = id;
        this.row = row;
        this.column = column;
        this.type = type;
        this.status = SeatStatus.AVAILABLE;
    }

    public String getId() { return id; }
    public int getRow() { return row; }
    public int getColumn() { return column; }
    public SeatType getType() { return type; }
    public SeatStatus getStatus() { return status; }
    public void setStatus(SeatStatus status) { this.status = status; }
}

// ============ MOVIE ============
public class Movie {
    private final String id;
    private final String title;
    private final String description;
    private final int durationMinutes;
    private final String language;
    private final String genre;

    public Movie(String id, String title, String description,
                 int durationMinutes, String language, String genre) {
        this.id = id;
        this.title = title;
        this.description = description;
        this.durationMinutes = durationMinutes;
        this.language = language;
        this.genre = genre;
    }

    public String getId() { return id; }
    public String getTitle() { return title; }
    public int getDurationMinutes() { return durationMinutes; }
}

// ============ SCREEN ============
public class Screen {
    private final String id;
    private final String name;
    private final List<Seat> seats;
    private final int totalSeats;

    public Screen(String id, String name, int rows, int columns) {
        this.id = id;
        this.name = name;
        this.seats = new ArrayList<>();
        this.totalSeats = rows * columns;
        initializeSeats(rows, columns);
    }

    private void initializeSeats(int rows, int columns) {
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < columns; c++) {
                SeatType type = (r < 2) ? SeatType.VIP :
                               (r < 5) ? SeatType.PREMIUM : SeatType.REGULAR;
                String seatId = (char)('A' + r) + String.valueOf(c + 1);
                seats.add(new Seat(seatId, r, c, type));
            }
        }
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public List<Seat> getSeats() { return Collections.unmodifiableList(seats); }
}

// ============ SHOW ============
public class Show {
    private final String id;
    private final Movie movie;
    private final Screen screen;
    private final LocalDateTime startTime;
    private final LocalDateTime endTime;
    private final Map<String, Seat> seatMap;
    private final ReentrantLock lock = new ReentrantLock();

    public Show(String id, Movie movie, Screen screen, LocalDateTime startTime) {
        this.id = id;
        this.movie = movie;
        this.screen = screen;
        this.startTime = startTime;
        this.endTime = startTime.plusMinutes(movie.getDurationMinutes());
        this.seatMap = new ConcurrentHashMap<>();

        // Create show-specific seat instances
        for (Seat seat : screen.getSeats()) {
            seatMap.put(seat.getId(), new Seat(seat.getId(), seat.getRow(),
                       seat.getColumn(), seat.getType()));
        }
    }

    public List<Seat> getAvailableSeats() {
        return seatMap.values().stream()
            .filter(s -> s.getStatus() == SeatStatus.AVAILABLE)
            .collect(Collectors.toList());
    }

    public boolean blockSeats(List<String> seatIds) {
        lock.lock();
        try {
            // Verify all seats are available
            for (String seatId : seatIds) {
                Seat seat = seatMap.get(seatId);
                if (seat == null || seat.getStatus() != SeatStatus.AVAILABLE) {
                    return false;
                }
            }
            // Block all seats
            for (String seatId : seatIds) {
                seatMap.get(seatId).setStatus(SeatStatus.BLOCKED);
            }
            return true;
        } finally {
            lock.unlock();
        }
    }

    public void confirmSeats(List<String> seatIds) {
        lock.lock();
        try {
            for (String seatId : seatIds) {
                seatMap.get(seatId).setStatus(SeatStatus.BOOKED);
            }
        } finally {
            lock.unlock();
        }
    }

    public void releaseSeats(List<String> seatIds) {
        lock.lock();
        try {
            for (String seatId : seatIds) {
                seatMap.get(seatId).setStatus(SeatStatus.AVAILABLE);
            }
        } finally {
            lock.unlock();
        }
    }

    public double calculatePrice(List<String> seatIds) {
        return seatIds.stream()
            .map(seatMap::get)
            .filter(Objects::nonNull)
            .mapToDouble(s -> s.getType().getBasePrice())
            .sum();
    }

    public String getId() { return id; }
    public Movie getMovie() { return movie; }
    public Screen getScreen() { return screen; }
    public LocalDateTime getStartTime() { return startTime; }
}

// ============ THEATER ============
public class Theater {
    private final String id;
    private final String name;
    private final String address;
    private final String city;
    private final List<Screen> screens;
    private final List<Show> shows;

    public Theater(String id, String name, String address, String city) {
        this.id = id;
        this.name = name;
        this.address = address;
        this.city = city;
        this.screens = new ArrayList<>();
        this.shows = new ArrayList<>();
    }

    public void addScreen(Screen screen) {
        screens.add(screen);
    }

    public void addShow(Show show) {
        shows.add(show);
    }

    public List<Show> getShowsForMovie(String movieId) {
        return shows.stream()
            .filter(s -> s.getMovie().getId().equals(movieId))
            .collect(Collectors.toList());
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getCity() { return city; }
}

// ============ BOOKING ============
public class Booking {
    private final String id;
    private final Show show;
    private final List<String> seatIds;
    private final String userId;
    private final double totalAmount;
    private BookingStatus status;
    private final LocalDateTime createdAt;
    private LocalDateTime expiresAt;

    public Booking(String id, Show show, List<String> seatIds,
                   String userId, double totalAmount) {
        this.id = id;
        this.show = show;
        this.seatIds = new ArrayList<>(seatIds);
        this.userId = userId;
        this.totalAmount = totalAmount;
        this.status = BookingStatus.PENDING;
        this.createdAt = LocalDateTime.now();
        this.expiresAt = createdAt.plusMinutes(10); // 10 min to complete payment
    }

    public void confirm() {
        this.status = BookingStatus.CONFIRMED;
    }

    public void cancel() {
        this.status = BookingStatus.CANCELLED;
    }

    public void expire() {
        this.status = BookingStatus.EXPIRED;
    }

    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt) &&
               status == BookingStatus.PENDING;
    }

    public String getId() { return id; }
    public Show getShow() { return show; }
    public List<String> getSeatIds() { return Collections.unmodifiableList(seatIds); }
    public double getTotalAmount() { return totalAmount; }
    public BookingStatus getStatus() { return status; }
}

// ============ PAYMENT ============
public interface PaymentProcessor {
    boolean processPayment(String bookingId, double amount, String paymentMethod);
}

public class PaymentService implements PaymentProcessor {
    @Override
    public boolean processPayment(String bookingId, double amount, String paymentMethod) {
        // Simulate payment processing
        System.out.println("Processing payment of $" + amount +
                          " for booking " + bookingId + " via " + paymentMethod);
        return true; // Assume success
    }
}

// ============ BOOKING SERVICE ============
public class BookingService {
    private final Map<String, Booking> bookings = new ConcurrentHashMap<>();
    private final PaymentProcessor paymentProcessor;
    private final AtomicLong bookingCounter = new AtomicLong(0);
    private final ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(1);

    public BookingService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
        startExpirationChecker();
    }

    public Optional<Booking> createBooking(Show show, List<String> seatIds,
                                           String userId) {
        // Try to block seats
        if (!show.blockSeats(seatIds)) {
            return Optional.empty();
        }

        double amount = show.calculatePrice(seatIds);
        String bookingId = "BK" + bookingCounter.incrementAndGet();
        Booking booking = new Booking(bookingId, show, seatIds, userId, amount);
        bookings.put(bookingId, booking);

        return Optional.of(booking);
    }

    public boolean confirmBooking(String bookingId, String paymentMethod) {
        Booking booking = bookings.get(bookingId);
        if (booking == null || booking.getStatus() != BookingStatus.PENDING) {
            return false;
        }

        if (booking.isExpired()) {
            handleExpiredBooking(booking);
            return false;
        }

        // Process payment
        if (paymentProcessor.processPayment(bookingId,
            booking.getTotalAmount(), paymentMethod)) {
            booking.confirm();
            booking.getShow().confirmSeats(booking.getSeatIds());
            return true;
        }

        return false;
    }

    public boolean cancelBooking(String bookingId) {
        Booking booking = bookings.get(bookingId);
        if (booking == null) return false;

        booking.cancel();
        booking.getShow().releaseSeats(booking.getSeatIds());
        return true;
    }

    private void handleExpiredBooking(Booking booking) {
        booking.expire();
        booking.getShow().releaseSeats(booking.getSeatIds());
    }

    private void startExpirationChecker() {
        scheduler.scheduleAtFixedRate(() -> {
            bookings.values().stream()
                .filter(b -> b.getStatus() == BookingStatus.PENDING && b.isExpired())
                .forEach(this::handleExpiredBooking);
        }, 1, 1, TimeUnit.MINUTES);
    }
}

// ============ MOVIE BOOKING SYSTEM (Main) ============
public class MovieBookingSystem {
    private static MovieBookingSystem instance;

    private final Map<String, Movie> movies = new ConcurrentHashMap<>();
    private final Map<String, Theater> theaters = new ConcurrentHashMap<>();
    private final Map<String, List<Theater>> theatersByCity = new ConcurrentHashMap<>();
    private final BookingService bookingService;
    private final AtomicLong idCounter = new AtomicLong(0);

    private MovieBookingSystem() {
        this.bookingService = new BookingService(new PaymentService());
    }

    public static synchronized MovieBookingSystem getInstance() {
        if (instance == null) {
            instance = new MovieBookingSystem();
        }
        return instance;
    }

    public Movie addMovie(String title, String description,
                         int duration, String language, String genre) {
        String id = "M" + idCounter.incrementAndGet();
        Movie movie = new Movie(id, title, description, duration, language, genre);
        movies.put(id, movie);
        return movie;
    }

    public Theater addTheater(String name, String address, String city) {
        String id = "T" + idCounter.incrementAndGet();
        Theater theater = new Theater(id, name, address, city);
        theaters.put(id, theater);
        theatersByCity.computeIfAbsent(city, k -> new ArrayList<>()).add(theater);
        return theater;
    }

    public List<Theater> getTheatersInCity(String city) {
        return theatersByCity.getOrDefault(city, Collections.emptyList());
    }

    public List<Show> getShowsForMovie(String movieId, String city) {
        return getTheatersInCity(city).stream()
            .flatMap(t -> t.getShowsForMovie(movieId).stream())
            .collect(Collectors.toList());
    }

    public Optional<Booking> bookTickets(Show show, List<String> seatIds,
                                         String userId) {
        return bookingService.createBooking(show, seatIds, userId);
    }

    public boolean confirmBooking(String bookingId, String paymentMethod) {
        return bookingService.confirmBooking(bookingId, paymentMethod);
    }
}

// ============ DEMO ============
public class BookMyShowDemo {
    public static void main(String[] args) {
        MovieBookingSystem system = MovieBookingSystem.getInstance();

        // Add movie
        Movie movie = system.addMovie("Inception", "A mind-bending thriller",
                                      148, "English", "Sci-Fi");

        // Add theater
        Theater theater = system.addTheater("PVR Cinemas", "123 Main St", "Mumbai");
        Screen screen = new Screen("S1", "Screen 1", 10, 15);
        theater.addScreen(screen);

        // Add show
        Show show = new Show("SH1", movie, screen,
                            LocalDateTime.now().plusHours(2));
        theater.addShow(show);

        // Book tickets
        List<String> seatIds = List.of("A1", "A2", "A3");
        Optional<Booking> booking = system.bookTickets(show, seatIds, "user123");

        booking.ifPresent(b -> {
            System.out.println("Booking created: " + b.getId());
            System.out.println("Total: $" + b.getTotalAmount());

            // Confirm with payment
            if (system.confirmBooking(b.getId(), "CREDIT_CARD")) {
                System.out.println("Booking confirmed!");
            }
        });
    }
}
```

---

## Problem 2: Elevator System

### Requirements
- Multiple elevators in a building
- Efficient scheduling algorithm
- Handle concurrent requests
- Support different elevator states

### Code Implementation

```java
// ============ ENUMS ============
public enum Direction {
    UP, DOWN, IDLE
}

public enum ElevatorState {
    MOVING, STOPPED, IDLE, MAINTENANCE
}

public enum DoorState {
    OPEN, CLOSED, OPENING, CLOSING
}

// ============ REQUEST ============
public class Request {
    private final int floor;
    private final Direction direction;
    private final LocalDateTime timestamp;

    public Request(int floor, Direction direction) {
        this.floor = floor;
        this.direction = direction;
        this.timestamp = LocalDateTime.now();
    }

    public int getFloor() { return floor; }
    public Direction getDirection() { return direction; }
    public LocalDateTime getTimestamp() { return timestamp; }
}

// ============ ELEVATOR ============
public class Elevator {
    private final int id;
    private final int minFloor;
    private final int maxFloor;
    private final int capacity;

    private int currentFloor;
    private Direction direction;
    private ElevatorState state;
    private DoorState doorState;
    private final Set<Integer> destinationFloors;
    private final ReentrantLock lock = new ReentrantLock();

    public Elevator(int id, int minFloor, int maxFloor, int capacity) {
        this.id = id;
        this.minFloor = minFloor;
        this.maxFloor = maxFloor;
        this.capacity = capacity;
        this.currentFloor = minFloor;
        this.direction = Direction.IDLE;
        this.state = ElevatorState.IDLE;
        this.doorState = DoorState.CLOSED;
        this.destinationFloors = new TreeSet<>();
    }

    public void addDestination(int floor) {
        lock.lock();
        try {
            if (floor >= minFloor && floor <= maxFloor) {
                destinationFloors.add(floor);
                updateDirection();
            }
        } finally {
            lock.unlock();
        }
    }

    public void move() {
        lock.lock();
        try {
            if (destinationFloors.isEmpty()) {
                state = ElevatorState.IDLE;
                direction = Direction.IDLE;
                return;
            }

            state = ElevatorState.MOVING;

            if (direction == Direction.UP) {
                currentFloor++;
            } else if (direction == Direction.DOWN) {
                currentFloor--;
            }

            System.out.println("Elevator " + id + " at floor " + currentFloor);

            // Check if we should stop
            if (destinationFloors.contains(currentFloor)) {
                stop();
                destinationFloors.remove(currentFloor);
            }

            updateDirection();
        } finally {
            lock.unlock();
        }
    }

    private void stop() {
        state = ElevatorState.STOPPED;
        openDoors();
        // Simulate passenger boarding
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        closeDoors();
    }

    private void openDoors() {
        doorState = DoorState.OPENING;
        System.out.println("Elevator " + id + " doors opening at floor " + currentFloor);
        doorState = DoorState.OPEN;
    }

    private void closeDoors() {
        doorState = DoorState.CLOSING;
        System.out.println("Elevator " + id + " doors closing");
        doorState = DoorState.CLOSED;
    }

    private void updateDirection() {
        if (destinationFloors.isEmpty()) {
            direction = Direction.IDLE;
            return;
        }

        int nextFloor = getNextDestination();
        if (nextFloor > currentFloor) {
            direction = Direction.UP;
        } else if (nextFloor < currentFloor) {
            direction = Direction.DOWN;
        }
    }

    private int getNextDestination() {
        if (direction == Direction.UP) {
            // Find next floor above current
            for (int floor : destinationFloors) {
                if (floor >= currentFloor) return floor;
            }
        } else if (direction == Direction.DOWN) {
            // Find next floor below current
            Integer[] floors = destinationFloors.toArray(new Integer[0]);
            for (int i = floors.length - 1; i >= 0; i--) {
                if (floors[i] <= currentFloor) return floors[i];
            }
        }
        // Return first destination
        return destinationFloors.iterator().next();
    }

    public int distanceToFloor(int floor) {
        return Math.abs(currentFloor - floor);
    }

    public boolean isMovingTowards(int floor, Direction requestDirection) {
        if (state == ElevatorState.IDLE) return true;
        if (direction == Direction.IDLE) return true;

        if (direction == Direction.UP && floor >= currentFloor &&
            requestDirection == Direction.UP) {
            return true;
        }
        if (direction == Direction.DOWN && floor <= currentFloor &&
            requestDirection == Direction.DOWN) {
            return true;
        }
        return false;
    }

    public int getId() { return id; }
    public int getCurrentFloor() { return currentFloor; }
    public Direction getDirection() { return direction; }
    public ElevatorState getState() { return state; }
    public boolean hasDestinations() { return !destinationFloors.isEmpty(); }
}

// ============ ELEVATOR SCHEDULING STRATEGY ============
public interface ElevatorSchedulingStrategy {
    Elevator selectElevator(List<Elevator> elevators, Request request);
}

// LOOK Algorithm (SCAN variant)
public class LookSchedulingStrategy implements ElevatorSchedulingStrategy {
    @Override
    public Elevator selectElevator(List<Elevator> elevators, Request request) {
        Elevator best = null;
        int minScore = Integer.MAX_VALUE;

        for (Elevator elevator : elevators) {
            if (elevator.getState() == ElevatorState.MAINTENANCE) continue;

            int score = calculateScore(elevator, request);
            if (score < minScore) {
                minScore = score;
                best = elevator;
            }
        }
        return best;
    }

    private int calculateScore(Elevator elevator, Request request) {
        int distance = elevator.distanceToFloor(request.getFloor());

        // Prefer elevators moving towards the request
        if (elevator.isMovingTowards(request.getFloor(), request.getDirection())) {
            return distance;
        }
        // Penalize elevators moving away
        return distance + 1000;
    }
}

// ============ ELEVATOR CONTROLLER ============
public class ElevatorController {
    private final List<Elevator> elevators;
    private final ElevatorSchedulingStrategy strategy;
    private final BlockingQueue<Request> pendingRequests;
    private final ExecutorService executor;
    private volatile boolean running;

    public ElevatorController(int numElevators, int minFloor, int maxFloor,
                             ElevatorSchedulingStrategy strategy) {
        this.elevators = new ArrayList<>();
        this.strategy = strategy;
        this.pendingRequests = new LinkedBlockingQueue<>();
        this.executor = Executors.newFixedThreadPool(numElevators + 1);
        this.running = true;

        for (int i = 0; i < numElevators; i++) {
            elevators.add(new Elevator(i + 1, minFloor, maxFloor, 10));
        }

        startElevators();
        startRequestProcessor();
    }

    public void requestElevator(int floor, Direction direction) {
        Request request = new Request(floor, direction);
        pendingRequests.offer(request);
        System.out.println("Request received: Floor " + floor + ", Direction " + direction);
    }

    public void requestFloor(int elevatorId, int floor) {
        elevators.stream()
            .filter(e -> e.getId() == elevatorId)
            .findFirst()
            .ifPresent(e -> e.addDestination(floor));
    }

    private void startElevators() {
        for (Elevator elevator : elevators) {
            executor.submit(() -> {
                while (running) {
                    if (elevator.hasDestinations()) {
                        elevator.move();
                    }
                    try {
                        Thread.sleep(1000); // Move every second
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }

    private void startRequestProcessor() {
        executor.submit(() -> {
            while (running) {
                try {
                    Request request = pendingRequests.poll(1, TimeUnit.SECONDS);
                    if (request != null) {
                        Elevator elevator = strategy.selectElevator(elevators, request);
                        if (elevator != null) {
                            elevator.addDestination(request.getFloor());
                            System.out.println("Assigned Elevator " + elevator.getId() +
                                             " to floor " + request.getFloor());
                        }
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        });
    }

    public void shutdown() {
        running = false;
        executor.shutdown();
    }

    public List<Elevator> getElevators() {
        return Collections.unmodifiableList(elevators);
    }
}

// ============ BUILDING ============
public class Building {
    private final String name;
    private final int totalFloors;
    private final ElevatorController controller;

    public Building(String name, int totalFloors, int numElevators) {
        this.name = name;
        this.totalFloors = totalFloors;
        this.controller = new ElevatorController(
            numElevators, 0, totalFloors, new LookSchedulingStrategy()
        );
    }

    public void callElevator(int floor, Direction direction) {
        controller.requestElevator(floor, direction);
    }

    public void selectFloor(int elevatorId, int floor) {
        controller.requestFloor(elevatorId, floor);
    }

    public ElevatorController getController() { return controller; }
}

// ============ DEMO ============
public class ElevatorSystemDemo {
    public static void main(String[] args) throws InterruptedException {
        Building building = new Building("Tech Tower", 20, 3);

        // Simulate elevator requests
        building.callElevator(5, Direction.UP);
        building.callElevator(10, Direction.DOWN);
        building.callElevator(1, Direction.UP);

        Thread.sleep(3000);

        // Internal floor selection
        building.selectFloor(1, 15);
        building.selectFloor(2, 3);

        Thread.sleep(10000);
        building.getController().shutdown();
    }
}
```

---

## Problem 3: Library Management System

### Requirements
- Manage books, members, librarians
- Book checkout and return
- Reservation system
- Fine calculation for late returns
- Search functionality

### Code Implementation

```java
// ============ ENUMS ============
public enum BookStatus {
    AVAILABLE, CHECKED_OUT, RESERVED, LOST
}

public enum MemberStatus {
    ACTIVE, SUSPENDED, CLOSED
}

public enum ReservationStatus {
    PENDING, FULFILLED, CANCELLED, EXPIRED
}

// ============ BOOK ============
public class Book {
    private final String isbn;
    private final String title;
    private final String author;
    private final String publisher;
    private final int publicationYear;
    private final String category;

    public Book(String isbn, String title, String author,
                String publisher, int publicationYear, String category) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.publisher = publisher;
        this.publicationYear = publicationYear;
        this.category = category;
    }

    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
    public String getCategory() { return category; }
}

// ============ BOOK ITEM (Physical Copy) ============
public class BookItem {
    private final String barcode;
    private final Book book;
    private BookStatus status;
    private LocalDate dueDate;
    private final LocalDate dateAdded;

    public BookItem(String barcode, Book book) {
        this.barcode = barcode;
        this.book = book;
        this.status = BookStatus.AVAILABLE;
        this.dateAdded = LocalDate.now();
    }

    public boolean checkout(int loanDays) {
        if (status != BookStatus.AVAILABLE) {
            return false;
        }
        status = BookStatus.CHECKED_OUT;
        dueDate = LocalDate.now().plusDays(loanDays);
        return true;
    }

    public void returnBook() {
        status = BookStatus.AVAILABLE;
        dueDate = null;
    }

    public boolean isOverdue() {
        return dueDate != null && LocalDate.now().isAfter(dueDate);
    }

    public long getOverdueDays() {
        if (dueDate == null) return 0;
        long days = ChronoUnit.DAYS.between(dueDate, LocalDate.now());
        return Math.max(0, days);
    }

    public String getBarcode() { return barcode; }
    public Book getBook() { return book; }
    public BookStatus getStatus() { return status; }
    public void setStatus(BookStatus status) { this.status = status; }
    public LocalDate getDueDate() { return dueDate; }
}

// ============ MEMBER ============
public class Member {
    private final String id;
    private final String name;
    private final String email;
    private final String phone;
    private MemberStatus status;
    private final LocalDate memberSince;
    private final List<Loan> currentLoans;
    private final List<Reservation> reservations;
    private double fineBalance;
    private static final int MAX_BOOKS = 5;

    public Member(String id, String name, String email, String phone) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.status = MemberStatus.ACTIVE;
        this.memberSince = LocalDate.now();
        this.currentLoans = new ArrayList<>();
        this.reservations = new ArrayList<>();
        this.fineBalance = 0;
    }

    public boolean canCheckout() {
        return status == MemberStatus.ACTIVE &&
               currentLoans.size() < MAX_BOOKS &&
               fineBalance < 10.0;
    }

    public void addLoan(Loan loan) {
        currentLoans.add(loan);
    }

    public void removeLoan(Loan loan) {
        currentLoans.remove(loan);
    }

    public void addFine(double amount) {
        fineBalance += amount;
    }

    public void payFine(double amount) {
        fineBalance = Math.max(0, fineBalance - amount);
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public MemberStatus getStatus() { return status; }
    public List<Loan> getCurrentLoans() { return Collections.unmodifiableList(currentLoans); }
    public double getFineBalance() { return fineBalance; }
}

// ============ LOAN ============
public class Loan {
    private final String id;
    private final BookItem bookItem;
    private final Member member;
    private final LocalDate loanDate;
    private final LocalDate dueDate;
    private LocalDate returnDate;
    private double fineAmount;

    public Loan(String id, BookItem bookItem, Member member, int loanDays) {
        this.id = id;
        this.bookItem = bookItem;
        this.member = member;
        this.loanDate = LocalDate.now();
        this.dueDate = loanDate.plusDays(loanDays);
    }

    public void completeReturn() {
        this.returnDate = LocalDate.now();
        if (returnDate.isAfter(dueDate)) {
            long overdueDays = ChronoUnit.DAYS.between(dueDate, returnDate);
            this.fineAmount = overdueDays * 0.50; // $0.50 per day
        }
    }

    public String getId() { return id; }
    public BookItem getBookItem() { return bookItem; }
    public Member getMember() { return member; }
    public LocalDate getDueDate() { return dueDate; }
    public double getFineAmount() { return fineAmount; }
}

// ============ RESERVATION ============
public class Reservation {
    private final String id;
    private final Book book;
    private final Member member;
    private final LocalDateTime createdAt;
    private ReservationStatus status;

    public Reservation(String id, Book book, Member member) {
        this.id = id;
        this.book = book;
        this.member = member;
        this.createdAt = LocalDateTime.now();
        this.status = ReservationStatus.PENDING;
    }

    public void fulfill() { this.status = ReservationStatus.FULFILLED; }
    public void cancel() { this.status = ReservationStatus.CANCELLED; }
    public void expire() { this.status = ReservationStatus.EXPIRED; }

    public String getId() { return id; }
    public Book getBook() { return book; }
    public Member getMember() { return member; }
    public ReservationStatus getStatus() { return status; }
}

// ============ LIBRARY CATALOG ============
public class Catalog {
    private final Map<String, Book> booksByIsbn = new ConcurrentHashMap<>();
    private final Map<String, BookItem> bookItemsByBarcode = new ConcurrentHashMap<>();
    private final Map<String, List<BookItem>> bookItemsByIsbn = new ConcurrentHashMap<>();

    public void addBook(Book book) {
        booksByIsbn.put(book.getIsbn(), book);
        bookItemsByIsbn.putIfAbsent(book.getIsbn(), new ArrayList<>());
    }

    public void addBookItem(BookItem bookItem) {
        bookItemsByBarcode.put(bookItem.getBarcode(), bookItem);
        bookItemsByIsbn.computeIfAbsent(bookItem.getBook().getIsbn(),
            k -> new ArrayList<>()).add(bookItem);
    }

    public List<Book> searchByTitle(String title) {
        return booksByIsbn.values().stream()
            .filter(b -> b.getTitle().toLowerCase().contains(title.toLowerCase()))
            .collect(Collectors.toList());
    }

    public List<Book> searchByAuthor(String author) {
        return booksByIsbn.values().stream()
            .filter(b -> b.getAuthor().toLowerCase().contains(author.toLowerCase()))
            .collect(Collectors.toList());
    }

    public List<Book> searchByCategory(String category) {
        return booksByIsbn.values().stream()
            .filter(b -> b.getCategory().equalsIgnoreCase(category))
            .collect(Collectors.toList());
    }

    public Optional<BookItem> findAvailableCopy(String isbn) {
        return bookItemsByIsbn.getOrDefault(isbn, Collections.emptyList())
            .stream()
            .filter(bi -> bi.getStatus() == BookStatus.AVAILABLE)
            .findFirst();
    }

    public BookItem getBookItem(String barcode) {
        return bookItemsByBarcode.get(barcode);
    }

    public Book getBook(String isbn) {
        return booksByIsbn.get(isbn);
    }
}

// ============ LIBRARY SERVICE ============
public class LibraryService {
    private final Catalog catalog;
    private final Map<String, Member> members = new ConcurrentHashMap<>();
    private final Map<String, Loan> activeLoans = new ConcurrentHashMap<>();
    private final Map<String, Queue<Reservation>> reservations = new ConcurrentHashMap<>();
    private final AtomicLong idCounter = new AtomicLong(0);
    private static final int LOAN_DAYS = 14;

    public LibraryService() {
        this.catalog = new Catalog();
    }

    public Member registerMember(String name, String email, String phone) {
        String id = "M" + idCounter.incrementAndGet();
        Member member = new Member(id, name, email, phone);
        members.put(id, member);
        return member;
    }

    public Optional<Loan> checkoutBook(String memberId, String isbn) {
        Member member = members.get(memberId);
        if (member == null || !member.canCheckout()) {
            return Optional.empty();
        }

        Optional<BookItem> bookItemOpt = catalog.findAvailableCopy(isbn);
        if (bookItemOpt.isEmpty()) {
            return Optional.empty();
        }

        BookItem bookItem = bookItemOpt.get();
        if (!bookItem.checkout(LOAN_DAYS)) {
            return Optional.empty();
        }

        String loanId = "L" + idCounter.incrementAndGet();
        Loan loan = new Loan(loanId, bookItem, member, LOAN_DAYS);
        member.addLoan(loan);
        activeLoans.put(loanId, loan);

        System.out.println("Book checked out: " + bookItem.getBook().getTitle() +
                          " to " + member.getName() + ", Due: " + loan.getDueDate());

        return Optional.of(loan);
    }

    public double returnBook(String barcode) {
        BookItem bookItem = catalog.getBookItem(barcode);
        if (bookItem == null || bookItem.getStatus() != BookStatus.CHECKED_OUT) {
            return 0;
        }

        // Find the loan
        Optional<Loan> loanOpt = activeLoans.values().stream()
            .filter(l -> l.getBookItem().getBarcode().equals(barcode))
            .findFirst();

        if (loanOpt.isEmpty()) {
            return 0;
        }

        Loan loan = loanOpt.get();
        loan.completeReturn();

        double fine = loan.getFineAmount();
        if (fine > 0) {
            loan.getMember().addFine(fine);
            System.out.println("Late return fine: $" + fine);
        }

        loan.getMember().removeLoan(loan);
        activeLoans.remove(loan.getId());
        bookItem.returnBook();

        // Check for pending reservations
        checkReservations(bookItem.getBook().getIsbn());

        System.out.println("Book returned: " + bookItem.getBook().getTitle());
        return fine;
    }

    public Optional<Reservation> reserveBook(String memberId, String isbn) {
        Member member = members.get(memberId);
        Book book = catalog.getBook(isbn);

        if (member == null || book == null) {
            return Optional.empty();
        }

        String reservationId = "R" + idCounter.incrementAndGet();
        Reservation reservation = new Reservation(reservationId, book, member);

        reservations.computeIfAbsent(isbn, k -> new LinkedList<>()).offer(reservation);

        System.out.println("Reservation created for: " + book.getTitle());
        return Optional.of(reservation);
    }

    private void checkReservations(String isbn) {
        Queue<Reservation> queue = reservations.get(isbn);
        if (queue != null && !queue.isEmpty()) {
            Reservation reservation = queue.peek();
            // Notify member that book is available
            System.out.println("Notification: Book available for " +
                             reservation.getMember().getName());
        }
    }

    public List<Book> searchBooks(String query) {
        Set<Book> results = new HashSet<>();
        results.addAll(catalog.searchByTitle(query));
        results.addAll(catalog.searchByAuthor(query));
        return new ArrayList<>(results);
    }

    public Catalog getCatalog() { return catalog; }
}

// ============ DEMO ============
public class LibraryManagementDemo {
    public static void main(String[] args) {
        LibraryService library = new LibraryService();

        // Add books
        Book book1 = new Book("978-0-13-468599-1", "Clean Code",
                             "Robert C. Martin", "Prentice Hall", 2008, "Programming");
        Book book2 = new Book("978-0-596-51774-8", "JavaScript: The Good Parts",
                             "Douglas Crockford", "O'Reilly", 2008, "Programming");

        library.getCatalog().addBook(book1);
        library.getCatalog().addBook(book2);

        // Add book copies
        library.getCatalog().addBookItem(new BookItem("BC001", book1));
        library.getCatalog().addBookItem(new BookItem("BC002", book1));
        library.getCatalog().addBookItem(new BookItem("BC003", book2));

        // Register members
        Member member1 = library.registerMember("Alice", "alice@email.com", "1234567890");
        Member member2 = library.registerMember("Bob", "bob@email.com", "0987654321");

        // Checkout book
        library.checkoutBook(member1.getId(), book1.getIsbn());

        // Search books
        List<Book> results = library.searchBooks("Clean");
        System.out.println("Search results: " + results.size() + " books found");

        // Return book (simulating late return would add fine)
        library.returnBook("BC001");
    }
}
```

---

## Problem 4: ATM Machine

### Requirements
- Card authentication
- Balance inquiry
- Cash withdrawal
- Cash deposit
- Fund transfer
- State management

### Code Implementation

```java
// ============ ENUMS ============
public enum TransactionType {
    BALANCE_INQUIRY, WITHDRAWAL, DEPOSIT, TRANSFER
}

public enum TransactionStatus {
    SUCCESS, FAILED, CANCELLED
}

// ============ CARD ============
public class Card {
    private final String cardNumber;
    private final String accountId;
    private final LocalDate expiryDate;
    private int pin;
    private int wrongPinAttempts;
    private boolean blocked;

    public Card(String cardNumber, String accountId, LocalDate expiryDate, int pin) {
        this.cardNumber = cardNumber;
        this.accountId = accountId;
        this.expiryDate = expiryDate;
        this.pin = pin;
        this.wrongPinAttempts = 0;
        this.blocked = false;
    }

    public boolean validatePin(int enteredPin) {
        if (blocked) return false;

        if (pin == enteredPin) {
            wrongPinAttempts = 0;
            return true;
        }

        wrongPinAttempts++;
        if (wrongPinAttempts >= 3) {
            blocked = true;
            System.out.println("Card blocked due to too many wrong PIN attempts");
        }
        return false;
    }

    public boolean isExpired() {
        return LocalDate.now().isAfter(expiryDate);
    }

    public String getCardNumber() { return cardNumber; }
    public String getAccountId() { return accountId; }
    public boolean isBlocked() { return blocked; }
}

// ============ ACCOUNT ============
public class Account {
    private final String id;
    private final String holderName;
    private double balance;
    private final ReentrantLock lock = new ReentrantLock();

    public Account(String id, String holderName, double initialBalance) {
        this.id = id;
        this.holderName = holderName;
        this.balance = initialBalance;
    }

    public double getBalance() {
        lock.lock();
        try {
            return balance;
        } finally {
            lock.unlock();
        }
    }

    public boolean withdraw(double amount) {
        lock.lock();
        try {
            if (amount > balance) {
                return false;
            }
            balance -= amount;
            return true;
        } finally {
            lock.unlock();
        }
    }

    public void deposit(double amount) {
        lock.lock();
        try {
            balance += amount;
        } finally {
            lock.unlock();
        }
    }

    public String getId() { return id; }
    public String getHolderName() { return holderName; }
}

// ============ TRANSACTION ============
public class Transaction {
    private final String id;
    private final TransactionType type;
    private final String accountId;
    private final double amount;
    private final LocalDateTime timestamp;
    private TransactionStatus status;

    public Transaction(String id, TransactionType type, String accountId, double amount) {
        this.id = id;
        this.type = type;
        this.accountId = accountId;
        this.amount = amount;
        this.timestamp = LocalDateTime.now();
        this.status = TransactionStatus.SUCCESS;
    }

    public void setStatus(TransactionStatus status) { this.status = status; }

    public String getId() { return id; }
    public TransactionType getType() { return type; }
    public double getAmount() { return amount; }
    public TransactionStatus getStatus() { return status; }
    public LocalDateTime getTimestamp() { return timestamp; }
}

// ============ CASH DISPENSER ============
public class CashDispenser {
    private final Map<Integer, Integer> cashInventory; // denomination -> count
    private final ReentrantLock lock = new ReentrantLock();

    public CashDispenser() {
        cashInventory = new ConcurrentHashMap<>();
        // Initialize with some cash
        cashInventory.put(100, 100);  // 100 x $100 bills
        cashInventory.put(50, 200);   // 200 x $50 bills
        cashInventory.put(20, 500);   // 500 x $20 bills
        cashInventory.put(10, 500);   // 500 x $10 bills
    }

    public boolean canDispense(double amount) {
        return amount <= getTotalCash() && amount % 10 == 0;
    }

    public Map<Integer, Integer> dispense(double amount) {
        lock.lock();
        try {
            Map<Integer, Integer> dispensed = new LinkedHashMap<>();
            int remaining = (int) amount;

            // Greedy approach: use largest denominations first
            int[] denominations = {100, 50, 20, 10};

            for (int denom : denominations) {
                if (remaining >= denom && cashInventory.getOrDefault(denom, 0) > 0) {
                    int needed = remaining / denom;
                    int available = cashInventory.get(denom);
                    int toDispense = Math.min(needed, available);

                    if (toDispense > 0) {
                        dispensed.put(denom, toDispense);
                        remaining -= toDispense * denom;
                        cashInventory.put(denom, available - toDispense);
                    }
                }
            }

            if (remaining > 0) {
                // Rollback
                dispensed.forEach((denom, count) ->
                    cashInventory.merge(denom, count, Integer::sum));
                return null;
            }

            return dispensed;
        } finally {
            lock.unlock();
        }
    }

    public void addCash(int denomination, int count) {
        lock.lock();
        try {
            cashInventory.merge(denomination, count, Integer::sum);
        } finally {
            lock.unlock();
        }
    }

    public double getTotalCash() {
        return cashInventory.entrySet().stream()
            .mapToDouble(e -> e.getKey() * e.getValue())
            .sum();
    }
}

// ============ ATM STATE (State Pattern) ============
public interface ATMState {
    void insertCard(ATM atm, Card card);
    void enterPin(ATM atm, int pin);
    void selectTransaction(ATM atm, TransactionType type);
    void executeTransaction(ATM atm, double amount);
    void ejectCard(ATM atm);
}

public class IdleState implements ATMState {
    @Override
    public void insertCard(ATM atm, Card card) {
        if (card.isBlocked()) {
            System.out.println("Card is blocked. Please contact bank.");
            return;
        }
        if (card.isExpired()) {
            System.out.println("Card is expired.");
            return;
        }
        atm.setCurrentCard(card);
        atm.setState(new CardInsertedState());
        System.out.println("Card inserted. Please enter PIN.");
    }

    @Override
    public void enterPin(ATM atm, int pin) {
        System.out.println("Please insert card first.");
    }

    @Override
    public void selectTransaction(ATM atm, TransactionType type) {
        System.out.println("Please insert card first.");
    }

    @Override
    public void executeTransaction(ATM atm, double amount) {
        System.out.println("Please insert card first.");
    }

    @Override
    public void ejectCard(ATM atm) {
        System.out.println("No card to eject.");
    }
}

public class CardInsertedState implements ATMState {
    @Override
    public void insertCard(ATM atm, Card card) {
        System.out.println("Card already inserted.");
    }

    @Override
    public void enterPin(ATM atm, int pin) {
        if (atm.getCurrentCard().validatePin(pin)) {
            atm.setState(new AuthenticatedState());
            System.out.println("PIN verified. Select transaction.");
        } else {
            System.out.println("Wrong PIN. Try again.");
            if (atm.getCurrentCard().isBlocked()) {
                atm.ejectCard();
            }
        }
    }

    @Override
    public void selectTransaction(ATM atm, TransactionType type) {
        System.out.println("Please enter PIN first.");
    }

    @Override
    public void executeTransaction(ATM atm, double amount) {
        System.out.println("Please enter PIN first.");
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.setCurrentCard(null);
        atm.setState(new IdleState());
        System.out.println("Card ejected.");
    }
}

public class AuthenticatedState implements ATMState {
    @Override
    public void insertCard(ATM atm, Card card) {
        System.out.println("Card already inserted.");
    }

    @Override
    public void enterPin(ATM atm, int pin) {
        System.out.println("Already authenticated.");
    }

    @Override
    public void selectTransaction(ATM atm, TransactionType type) {
        atm.setSelectedTransaction(type);
        atm.setState(new TransactionSelectedState());
        System.out.println("Transaction selected: " + type);
    }

    @Override
    public void executeTransaction(ATM atm, double amount) {
        System.out.println("Please select transaction type first.");
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.setCurrentCard(null);
        atm.setState(new IdleState());
        System.out.println("Card ejected. Thank you!");
    }
}

public class TransactionSelectedState implements ATMState {
    @Override
    public void insertCard(ATM atm, Card card) {
        System.out.println("Card already inserted.");
    }

    @Override
    public void enterPin(ATM atm, int pin) {
        System.out.println("Already authenticated.");
    }

    @Override
    public void selectTransaction(ATM atm, TransactionType type) {
        atm.setSelectedTransaction(type);
        System.out.println("Transaction changed to: " + type);
    }

    @Override
    public void executeTransaction(ATM atm, double amount) {
        atm.processTransaction(amount);
        atm.setState(new AuthenticatedState());
    }

    @Override
    public void ejectCard(ATM atm) {
        atm.setCurrentCard(null);
        atm.setSelectedTransaction(null);
        atm.setState(new IdleState());
        System.out.println("Card ejected.");
    }
}

// ============ ATM ============
public class ATM {
    private final String id;
    private final String location;
    private ATMState state;
    private Card currentCard;
    private TransactionType selectedTransaction;
    private final CashDispenser cashDispenser;
    private final Bank bank;
    private final List<Transaction> transactions;
    private final AtomicLong transactionCounter = new AtomicLong(0);

    public ATM(String id, String location, Bank bank) {
        this.id = id;
        this.location = location;
        this.state = new IdleState();
        this.cashDispenser = new CashDispenser();
        this.bank = bank;
        this.transactions = new ArrayList<>();
    }

    public void insertCard(Card card) {
        state.insertCard(this, card);
    }

    public void enterPin(int pin) {
        state.enterPin(this, pin);
    }

    public void selectTransaction(TransactionType type) {
        state.selectTransaction(this, type);
    }

    public void executeTransaction(double amount) {
        state.executeTransaction(this, amount);
    }

    public void ejectCard() {
        state.ejectCard(this);
    }

    public void processTransaction(double amount) {
        Account account = bank.getAccount(currentCard.getAccountId());
        if (account == null) {
            System.out.println("Account not found.");
            return;
        }

        String txnId = "TXN" + transactionCounter.incrementAndGet();
        Transaction transaction = new Transaction(txnId, selectedTransaction,
                                                  account.getId(), amount);

        switch (selectedTransaction) {
            case BALANCE_INQUIRY:
                System.out.println("Current Balance: $" + account.getBalance());
                break;

            case WITHDRAWAL:
                if (!cashDispenser.canDispense(amount)) {
                    System.out.println("Cannot dispense this amount.");
                    transaction.setStatus(TransactionStatus.FAILED);
                } else if (account.withdraw(amount)) {
                    Map<Integer, Integer> cash = cashDispenser.dispense(amount);
                    System.out.println("Please collect your cash:");
                    cash.forEach((denom, count) ->
                        System.out.println("  $" + denom + " x " + count));
                    System.out.println("New Balance: $" + account.getBalance());
                } else {
                    System.out.println("Insufficient funds.");
                    transaction.setStatus(TransactionStatus.FAILED);
                }
                break;

            case DEPOSIT:
                account.deposit(amount);
                cashDispenser.addCash(20, (int)(amount / 20)); // Simplified
                System.out.println("Deposited: $" + amount);
                System.out.println("New Balance: $" + account.getBalance());
                break;

            default:
                System.out.println("Transaction type not supported.");
                transaction.setStatus(TransactionStatus.FAILED);
        }

        transactions.add(transaction);
    }

    public void setState(ATMState state) { this.state = state; }
    public Card getCurrentCard() { return currentCard; }
    public void setCurrentCard(Card card) { this.currentCard = card; }
    public void setSelectedTransaction(TransactionType type) {
        this.selectedTransaction = type;
    }
}

// ============ BANK ============
public class Bank {
    private final String name;
    private final Map<String, Account> accounts = new ConcurrentHashMap<>();
    private final Map<String, Card> cards = new ConcurrentHashMap<>();

    public Bank(String name) {
        this.name = name;
    }

    public Account createAccount(String holderName, double initialDeposit) {
        String id = "ACC" + System.currentTimeMillis();
        Account account = new Account(id, holderName, initialDeposit);
        accounts.put(id, account);
        return account;
    }

    public Card issueCard(String accountId, int pin) {
        String cardNumber = "4" + String.format("%015d", System.currentTimeMillis());
        Card card = new Card(cardNumber, accountId,
                            LocalDate.now().plusYears(3), pin);
        cards.put(cardNumber, card);
        return card;
    }

    public Account getAccount(String accountId) {
        return accounts.get(accountId);
    }

    public Card getCard(String cardNumber) {
        return cards.get(cardNumber);
    }
}

// ============ DEMO ============
public class ATMDemo {
    public static void main(String[] args) {
        // Setup bank and accounts
        Bank bank = new Bank("National Bank");
        Account account = bank.createAccount("John Doe", 5000.0);
        Card card = bank.issueCard(account.getId(), 1234);

        // Create ATM
        ATM atm = new ATM("ATM001", "123 Main Street", bank);

        // Simulate ATM usage
        System.out.println("=== ATM Session ===\n");

        // Insert card
        atm.insertCard(card);

        // Enter PIN (wrong first)
        atm.enterPin(9999);

        // Enter correct PIN
        atm.enterPin(1234);

        // Check balance
        atm.selectTransaction(TransactionType.BALANCE_INQUIRY);
        atm.executeTransaction(0);

        // Withdraw cash
        atm.selectTransaction(TransactionType.WITHDRAWAL);
        atm.executeTransaction(200);

        // Deposit cash
        atm.selectTransaction(TransactionType.DEPOSIT);
        atm.executeTransaction(500);

        // Final balance check
        atm.selectTransaction(TransactionType.BALANCE_INQUIRY);
        atm.executeTransaction(0);

        // Eject card
        atm.ejectCard();
    }
}
```

---

*Continued in Part 2...*
