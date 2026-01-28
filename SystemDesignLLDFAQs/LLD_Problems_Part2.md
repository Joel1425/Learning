# LLD Interview Problems - Part 2

## Problem 5: Tic-Tac-Toe

### Requirements
- 3x3 or NxN board
- Two players (X and O)
- Win detection (row, column, diagonal)
- Draw detection
- Extensible for different board sizes

### Code Implementation

```java
// ============ ENUMS ============
public enum PieceType {
    X('X'),
    O('O'),
    EMPTY(' ');

    private final char symbol;

    PieceType(char symbol) {
        this.symbol = symbol;
    }

    public char getSymbol() { return symbol; }
}

public enum GameStatus {
    IN_PROGRESS, X_WINS, O_WINS, DRAW
}

// ============ PLAYER ============
public class Player {
    private final String id;
    private final String name;
    private final PieceType piece;

    public Player(String id, String name, PieceType piece) {
        this.id = id;
        this.name = name;
        this.piece = piece;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public PieceType getPiece() { return piece; }
}

// ============ BOARD ============
public class Board {
    private final int size;
    private final PieceType[][] grid;
    private int movesCount;

    public Board(int size) {
        this.size = size;
        this.grid = new PieceType[size][size];
        this.movesCount = 0;
        initializeBoard();
    }

    private void initializeBoard() {
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                grid[i][j] = PieceType.EMPTY;
            }
        }
    }

    public boolean placePiece(int row, int col, PieceType piece) {
        if (!isValidPosition(row, col)) {
            return false;
        }
        if (grid[row][col] != PieceType.EMPTY) {
            return false;
        }
        grid[row][col] = piece;
        movesCount++;
        return true;
    }

    public boolean isValidPosition(int row, int col) {
        return row >= 0 && row < size && col >= 0 && col < size;
    }

    public boolean isFull() {
        return movesCount >= size * size;
    }

    public PieceType getPiece(int row, int col) {
        if (!isValidPosition(row, col)) return null;
        return grid[row][col];
    }

    public int getSize() { return size; }

    public void display() {
        System.out.println();
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                System.out.print(" " + grid[i][j].getSymbol() + " ");
                if (j < size - 1) System.out.print("|");
            }
            System.out.println();
            if (i < size - 1) {
                System.out.println("-".repeat(size * 4 - 1));
            }
        }
        System.out.println();
    }

    public void reset() {
        initializeBoard();
        movesCount = 0;
    }
}

// ============ WIN STRATEGY (Strategy Pattern) ============
public interface WinStrategy {
    boolean checkWin(Board board, int lastRow, int lastCol, PieceType piece);
}

public class StandardWinStrategy implements WinStrategy {
    @Override
    public boolean checkWin(Board board, int lastRow, int lastCol, PieceType piece) {
        int size = board.getSize();

        // Check row
        if (checkLine(board, lastRow, 0, 0, 1, piece)) return true;

        // Check column
        if (checkLine(board, 0, lastCol, 1, 0, piece)) return true;

        // Check main diagonal (if applicable)
        if (lastRow == lastCol) {
            if (checkLine(board, 0, 0, 1, 1, piece)) return true;
        }

        // Check anti-diagonal (if applicable)
        if (lastRow + lastCol == size - 1) {
            if (checkLine(board, 0, size - 1, 1, -1, piece)) return true;
        }

        return false;
    }

    private boolean checkLine(Board board, int startRow, int startCol,
                             int rowDelta, int colDelta, PieceType piece) {
        int size = board.getSize();
        int count = 0;

        for (int i = 0; i < size; i++) {
            int row = startRow + i * rowDelta;
            int col = startCol + i * colDelta;

            if (board.getPiece(row, col) == piece) {
                count++;
            }
        }

        return count == size;
    }
}

// ============ GAME ============
public class TicTacToeGame {
    private final Board board;
    private final Player player1;
    private final Player player2;
    private Player currentPlayer;
    private GameStatus status;
    private final WinStrategy winStrategy;

    public TicTacToeGame(int boardSize, Player player1, Player player2) {
        this.board = new Board(boardSize);
        this.player1 = player1;
        this.player2 = player2;
        this.currentPlayer = player1;
        this.status = GameStatus.IN_PROGRESS;
        this.winStrategy = new StandardWinStrategy();
    }

    public TicTacToeGame(Player player1, Player player2) {
        this(3, player1, player2); // Default 3x3
    }

    public boolean makeMove(int row, int col) {
        if (status != GameStatus.IN_PROGRESS) {
            System.out.println("Game is already over!");
            return false;
        }

        if (!board.placePiece(row, col, currentPlayer.getPiece())) {
            System.out.println("Invalid move! Try again.");
            return false;
        }

        System.out.println(currentPlayer.getName() + " placed " +
                          currentPlayer.getPiece() + " at (" + row + ", " + col + ")");

        // Check for win
        if (winStrategy.checkWin(board, row, col, currentPlayer.getPiece())) {
            status = (currentPlayer.getPiece() == PieceType.X) ?
                     GameStatus.X_WINS : GameStatus.O_WINS;
            System.out.println(currentPlayer.getName() + " wins!");
            return true;
        }

        // Check for draw
        if (board.isFull()) {
            status = GameStatus.DRAW;
            System.out.println("It's a draw!");
            return true;
        }

        // Switch player
        currentPlayer = (currentPlayer == player1) ? player2 : player1;
        return true;
    }

    public void displayBoard() {
        board.display();
    }

    public void reset() {
        board.reset();
        currentPlayer = player1;
        status = GameStatus.IN_PROGRESS;
        System.out.println("Game reset!");
    }

    public GameStatus getStatus() { return status; }
    public Player getCurrentPlayer() { return currentPlayer; }
    public boolean isGameOver() { return status != GameStatus.IN_PROGRESS; }
}

// ============ GAME CONTROLLER (For Multiple Games) ============
public class TicTacToeController {
    private final Map<String, TicTacToeGame> games = new ConcurrentHashMap<>();
    private final AtomicLong gameCounter = new AtomicLong(0);

    public String createGame(String player1Name, String player2Name) {
        String gameId = "GAME" + gameCounter.incrementAndGet();

        Player p1 = new Player("P1", player1Name, PieceType.X);
        Player p2 = new Player("P2", player2Name, PieceType.O);

        TicTacToeGame game = new TicTacToeGame(p1, p2);
        games.put(gameId, game);

        System.out.println("Game created: " + gameId);
        return gameId;
    }

    public boolean makeMove(String gameId, int row, int col) {
        TicTacToeGame game = games.get(gameId);
        if (game == null) {
            System.out.println("Game not found!");
            return false;
        }
        return game.makeMove(row, col);
    }

    public void displayBoard(String gameId) {
        TicTacToeGame game = games.get(gameId);
        if (game != null) {
            game.displayBoard();
        }
    }

    public TicTacToeGame getGame(String gameId) {
        return games.get(gameId);
    }
}

// ============ DEMO ============
public class TicTacToeDemo {
    public static void main(String[] args) {
        Player player1 = new Player("1", "Alice", PieceType.X);
        Player player2 = new Player("2", "Bob", PieceType.O);

        TicTacToeGame game = new TicTacToeGame(player1, player2);

        game.displayBoard();

        // Simulate a game
        game.makeMove(0, 0); // Alice: X at (0,0)
        game.displayBoard();

        game.makeMove(1, 1); // Bob: O at (1,1)
        game.displayBoard();

        game.makeMove(0, 1); // Alice: X at (0,1)
        game.displayBoard();

        game.makeMove(2, 2); // Bob: O at (2,2)
        game.displayBoard();

        game.makeMove(0, 2); // Alice: X at (0,2) - Alice wins!
        game.displayBoard();

        System.out.println("Game Status: " + game.getStatus());
    }
}
```

---

## Problem 6: Chess

### Requirements
- 8x8 board with all pieces
- Valid move rules for each piece
- Turn-based gameplay
- Check and checkmate detection
- Move validation

### Code Implementation

```java
// ============ ENUMS ============
public enum Color {
    WHITE, BLACK
}

public enum PieceType {
    KING, QUEEN, ROOK, BISHOP, KNIGHT, PAWN
}

// ============ POSITION ============
public class Position {
    private final int row;
    private final int col;

    public Position(int row, int col) {
        this.row = row;
        this.col = col;
    }

    public int getRow() { return row; }
    public int getCol() { return col; }

    public boolean isValid() {
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Position)) return false;
        Position position = (Position) o;
        return row == position.row && col == position.col;
    }

    @Override
    public int hashCode() {
        return Objects.hash(row, col);
    }

    @Override
    public String toString() {
        return (char)('a' + col) + String.valueOf(8 - row);
    }
}

// ============ PIECE (Abstract) ============
public abstract class Piece {
    protected final Color color;
    protected final PieceType type;
    protected Position position;
    protected boolean hasMoved;

    protected Piece(Color color, PieceType type, Position position) {
        this.color = color;
        this.type = type;
        this.position = position;
        this.hasMoved = false;
    }

    public abstract List<Position> getPossibleMoves(Board board);
    public abstract boolean canMove(Board board, Position to);

    protected boolean isValidMove(Position to) {
        return to.isValid();
    }

    protected boolean isPathClear(Board board, Position from, Position to) {
        int rowDiff = Integer.signum(to.getRow() - from.getRow());
        int colDiff = Integer.signum(to.getCol() - from.getCol());

        int currentRow = from.getRow() + rowDiff;
        int currentCol = from.getCol() + colDiff;

        while (currentRow != to.getRow() || currentCol != to.getCol()) {
            if (board.getPiece(new Position(currentRow, currentCol)) != null) {
                return false;
            }
            currentRow += rowDiff;
            currentCol += colDiff;
        }
        return true;
    }

    public Color getColor() { return color; }
    public PieceType getType() { return type; }
    public Position getPosition() { return position; }
    public void setPosition(Position position) {
        this.position = position;
        this.hasMoved = true;
    }
    public boolean hasMoved() { return hasMoved; }

    public char getSymbol() {
        char symbol = switch (type) {
            case KING -> 'K';
            case QUEEN -> 'Q';
            case ROOK -> 'R';
            case BISHOP -> 'B';
            case KNIGHT -> 'N';
            case PAWN -> 'P';
        };
        return color == Color.WHITE ? symbol : Character.toLowerCase(symbol);
    }
}

// ============ CONCRETE PIECES ============
public class King extends Piece {
    public King(Color color, Position position) {
        super(color, PieceType.KING, position);
    }

    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int[] directions = {-1, 0, 1};

        for (int dr : directions) {
            for (int dc : directions) {
                if (dr == 0 && dc == 0) continue;
                Position newPos = new Position(position.getRow() + dr,
                                               position.getCol() + dc);
                if (canMove(board, newPos)) {
                    moves.add(newPos);
                }
            }
        }
        return moves;
    }

    @Override
    public boolean canMove(Board board, Position to) {
        if (!isValidMove(to)) return false;

        int rowDiff = Math.abs(to.getRow() - position.getRow());
        int colDiff = Math.abs(to.getCol() - position.getCol());

        if (rowDiff > 1 || colDiff > 1) return false;

        Piece targetPiece = board.getPiece(to);
        return targetPiece == null || targetPiece.getColor() != this.color;
    }
}

public class Queen extends Piece {
    public Queen(Color color, Position position) {
        super(color, PieceType.QUEEN, position);
    }

    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        // Combine Rook and Bishop moves
        int[][] directions = {{0,1}, {0,-1}, {1,0}, {-1,0},
                             {1,1}, {1,-1}, {-1,1}, {-1,-1}};

        for (int[] dir : directions) {
            for (int i = 1; i < 8; i++) {
                Position newPos = new Position(position.getRow() + dir[0] * i,
                                               position.getCol() + dir[1] * i);
                if (!newPos.isValid()) break;

                Piece target = board.getPiece(newPos);
                if (target == null) {
                    moves.add(newPos);
                } else {
                    if (target.getColor() != this.color) {
                        moves.add(newPos);
                    }
                    break;
                }
            }
        }
        return moves;
    }

    @Override
    public boolean canMove(Board board, Position to) {
        if (!isValidMove(to)) return false;

        int rowDiff = Math.abs(to.getRow() - position.getRow());
        int colDiff = Math.abs(to.getCol() - position.getCol());

        boolean straightLine = (rowDiff == 0 || colDiff == 0);
        boolean diagonal = (rowDiff == colDiff);

        if (!straightLine && !diagonal) return false;
        if (!isPathClear(board, position, to)) return false;

        Piece targetPiece = board.getPiece(to);
        return targetPiece == null || targetPiece.getColor() != this.color;
    }
}

public class Rook extends Piece {
    public Rook(Color color, Position position) {
        super(color, PieceType.ROOK, position);
    }

    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int[][] directions = {{0,1}, {0,-1}, {1,0}, {-1,0}};

        for (int[] dir : directions) {
            for (int i = 1; i < 8; i++) {
                Position newPos = new Position(position.getRow() + dir[0] * i,
                                               position.getCol() + dir[1] * i);
                if (!newPos.isValid()) break;

                Piece target = board.getPiece(newPos);
                if (target == null) {
                    moves.add(newPos);
                } else {
                    if (target.getColor() != this.color) {
                        moves.add(newPos);
                    }
                    break;
                }
            }
        }
        return moves;
    }

    @Override
    public boolean canMove(Board board, Position to) {
        if (!isValidMove(to)) return false;

        int rowDiff = Math.abs(to.getRow() - position.getRow());
        int colDiff = Math.abs(to.getCol() - position.getCol());

        if (rowDiff != 0 && colDiff != 0) return false;
        if (!isPathClear(board, position, to)) return false;

        Piece targetPiece = board.getPiece(to);
        return targetPiece == null || targetPiece.getColor() != this.color;
    }
}

public class Bishop extends Piece {
    public Bishop(Color color, Position position) {
        super(color, PieceType.BISHOP, position);
    }

    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int[][] directions = {{1,1}, {1,-1}, {-1,1}, {-1,-1}};

        for (int[] dir : directions) {
            for (int i = 1; i < 8; i++) {
                Position newPos = new Position(position.getRow() + dir[0] * i,
                                               position.getCol() + dir[1] * i);
                if (!newPos.isValid()) break;

                Piece target = board.getPiece(newPos);
                if (target == null) {
                    moves.add(newPos);
                } else {
                    if (target.getColor() != this.color) {
                        moves.add(newPos);
                    }
                    break;
                }
            }
        }
        return moves;
    }

    @Override
    public boolean canMove(Board board, Position to) {
        if (!isValidMove(to)) return false;

        int rowDiff = Math.abs(to.getRow() - position.getRow());
        int colDiff = Math.abs(to.getCol() - position.getCol());

        if (rowDiff != colDiff) return false;
        if (!isPathClear(board, position, to)) return false;

        Piece targetPiece = board.getPiece(to);
        return targetPiece == null || targetPiece.getColor() != this.color;
    }
}

public class Knight extends Piece {
    public Knight(Color color, Position position) {
        super(color, PieceType.KNIGHT, position);
    }

    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int[][] jumps = {{-2,-1}, {-2,1}, {-1,-2}, {-1,2},
                        {1,-2}, {1,2}, {2,-1}, {2,1}};

        for (int[] jump : jumps) {
            Position newPos = new Position(position.getRow() + jump[0],
                                           position.getCol() + jump[1]);
            if (canMove(board, newPos)) {
                moves.add(newPos);
            }
        }
        return moves;
    }

    @Override
    public boolean canMove(Board board, Position to) {
        if (!isValidMove(to)) return false;

        int rowDiff = Math.abs(to.getRow() - position.getRow());
        int colDiff = Math.abs(to.getCol() - position.getCol());

        if (!((rowDiff == 2 && colDiff == 1) || (rowDiff == 1 && colDiff == 2))) {
            return false;
        }

        Piece targetPiece = board.getPiece(to);
        return targetPiece == null || targetPiece.getColor() != this.color;
    }
}

public class Pawn extends Piece {
    public Pawn(Color color, Position position) {
        super(color, PieceType.PAWN, position);
    }

    @Override
    public List<Position> getPossibleMoves(Board board) {
        List<Position> moves = new ArrayList<>();
        int direction = (color == Color.WHITE) ? -1 : 1;

        // Forward move
        Position forward = new Position(position.getRow() + direction, position.getCol());
        if (forward.isValid() && board.getPiece(forward) == null) {
            moves.add(forward);

            // Double move from starting position
            if (!hasMoved) {
                Position doubleForward = new Position(position.getRow() + 2 * direction,
                                                      position.getCol());
                if (board.getPiece(doubleForward) == null) {
                    moves.add(doubleForward);
                }
            }
        }

        // Diagonal captures
        for (int dc : new int[]{-1, 1}) {
            Position capture = new Position(position.getRow() + direction,
                                           position.getCol() + dc);
            if (capture.isValid()) {
                Piece target = board.getPiece(capture);
                if (target != null && target.getColor() != this.color) {
                    moves.add(capture);
                }
            }
        }

        return moves;
    }

    @Override
    public boolean canMove(Board board, Position to) {
        if (!isValidMove(to)) return false;

        int direction = (color == Color.WHITE) ? -1 : 1;
        int rowDiff = to.getRow() - position.getRow();
        int colDiff = Math.abs(to.getCol() - position.getCol());

        // Forward move
        if (colDiff == 0) {
            if (rowDiff == direction && board.getPiece(to) == null) {
                return true;
            }
            if (!hasMoved && rowDiff == 2 * direction) {
                Position middle = new Position(position.getRow() + direction,
                                              position.getCol());
                return board.getPiece(to) == null && board.getPiece(middle) == null;
            }
        }

        // Diagonal capture
        if (colDiff == 1 && rowDiff == direction) {
            Piece target = board.getPiece(to);
            return target != null && target.getColor() != this.color;
        }

        return false;
    }
}

// ============ BOARD ============
public class Board {
    private final Piece[][] grid;
    private final List<Piece> whitePieces;
    private final List<Piece> blackPieces;

    public Board() {
        grid = new Piece[8][8];
        whitePieces = new ArrayList<>();
        blackPieces = new ArrayList<>();
        initializeBoard();
    }

    private void initializeBoard() {
        // Place pawns
        for (int col = 0; col < 8; col++) {
            placePiece(new Pawn(Color.BLACK, new Position(1, col)));
            placePiece(new Pawn(Color.WHITE, new Position(6, col)));
        }

        // Place other pieces
        placePiecesForColor(Color.BLACK, 0);
        placePiecesForColor(Color.WHITE, 7);
    }

    private void placePiecesForColor(Color color, int row) {
        placePiece(new Rook(color, new Position(row, 0)));
        placePiece(new Knight(color, new Position(row, 1)));
        placePiece(new Bishop(color, new Position(row, 2)));
        placePiece(new Queen(color, new Position(row, 3)));
        placePiece(new King(color, new Position(row, 4)));
        placePiece(new Bishop(color, new Position(row, 5)));
        placePiece(new Knight(color, new Position(row, 6)));
        placePiece(new Rook(color, new Position(row, 7)));
    }

    private void placePiece(Piece piece) {
        Position pos = piece.getPosition();
        grid[pos.getRow()][pos.getCol()] = piece;

        if (piece.getColor() == Color.WHITE) {
            whitePieces.add(piece);
        } else {
            blackPieces.add(piece);
        }
    }

    public Piece getPiece(Position position) {
        if (!position.isValid()) return null;
        return grid[position.getRow()][position.getCol()];
    }

    public boolean movePiece(Position from, Position to) {
        Piece piece = getPiece(from);
        if (piece == null) return false;

        Piece captured = getPiece(to);
        if (captured != null) {
            if (captured.getColor() == Color.WHITE) {
                whitePieces.remove(captured);
            } else {
                blackPieces.remove(captured);
            }
        }

        grid[from.getRow()][from.getCol()] = null;
        grid[to.getRow()][to.getCol()] = piece;
        piece.setPosition(to);

        return true;
    }

    public Position findKing(Color color) {
        List<Piece> pieces = (color == Color.WHITE) ? whitePieces : blackPieces;
        return pieces.stream()
            .filter(p -> p.getType() == PieceType.KING)
            .findFirst()
            .map(Piece::getPosition)
            .orElse(null);
    }

    public boolean isUnderAttack(Position position, Color byColor) {
        List<Piece> attackingPieces = (byColor == Color.WHITE) ? whitePieces : blackPieces;
        return attackingPieces.stream()
            .anyMatch(p -> p.canMove(this, position));
    }

    public void display() {
        System.out.println("\n  a b c d e f g h");
        System.out.println("  ----------------");
        for (int row = 0; row < 8; row++) {
            System.out.print((8 - row) + "|");
            for (int col = 0; col < 8; col++) {
                Piece piece = grid[row][col];
                char symbol = (piece == null) ? '.' : piece.getSymbol();
                System.out.print(symbol + " ");
            }
            System.out.println("|" + (8 - row));
        }
        System.out.println("  ----------------");
        System.out.println("  a b c d e f g h\n");
    }
}

// ============ GAME ============
public class ChessGame {
    private final Board board;
    private Color currentTurn;
    private boolean isGameOver;
    private Color winner;
    private final List<String> moveHistory;

    public ChessGame() {
        this.board = new Board();
        this.currentTurn = Color.WHITE;
        this.isGameOver = false;
        this.moveHistory = new ArrayList<>();
    }

    public boolean makeMove(Position from, Position to) {
        if (isGameOver) {
            System.out.println("Game is over!");
            return false;
        }

        Piece piece = board.getPiece(from);
        if (piece == null) {
            System.out.println("No piece at " + from);
            return false;
        }

        if (piece.getColor() != currentTurn) {
            System.out.println("It's " + currentTurn + "'s turn!");
            return false;
        }

        if (!piece.canMove(board, to)) {
            System.out.println("Invalid move for " + piece.getType());
            return false;
        }

        // Make the move
        Piece captured = board.getPiece(to);
        board.movePiece(from, to);

        // Check if move puts own king in check
        if (isInCheck(currentTurn)) {
            // Undo move
            board.movePiece(to, from);
            System.out.println("Cannot move into check!");
            return false;
        }

        // Record move
        String notation = piece.getType() + " " + from + " -> " + to;
        if (captured != null) {
            notation += " x" + captured.getType();
        }
        moveHistory.add(notation);
        System.out.println(currentTurn + ": " + notation);

        // Check for checkmate
        Color opponent = (currentTurn == Color.WHITE) ? Color.BLACK : Color.WHITE;
        if (isInCheck(opponent)) {
            System.out.println(opponent + " is in CHECK!");
            if (isCheckmate(opponent)) {
                isGameOver = true;
                winner = currentTurn;
                System.out.println("CHECKMATE! " + currentTurn + " wins!");
            }
        }

        // Switch turn
        currentTurn = opponent;
        return true;
    }

    public boolean isInCheck(Color color) {
        Position kingPos = board.findKing(color);
        if (kingPos == null) return false;

        Color opponent = (color == Color.WHITE) ? Color.BLACK : Color.WHITE;
        return board.isUnderAttack(kingPos, opponent);
    }

    public boolean isCheckmate(Color color) {
        // Simplified checkmate detection
        // In full implementation, check if any move can escape check
        return isInCheck(color);
    }

    public void displayBoard() {
        board.display();
    }

    public Color getCurrentTurn() { return currentTurn; }
    public boolean isGameOver() { return isGameOver; }
    public Color getWinner() { return winner; }
}

// ============ DEMO ============
public class ChessDemo {
    public static void main(String[] args) {
        ChessGame game = new ChessGame();
        game.displayBoard();

        // Scholar's Mate example moves
        game.makeMove(new Position(6, 4), new Position(4, 4)); // e4
        game.displayBoard();

        game.makeMove(new Position(1, 4), new Position(3, 4)); // e5
        game.displayBoard();

        game.makeMove(new Position(7, 3), new Position(3, 7)); // Qh5
        game.displayBoard();

        game.makeMove(new Position(1, 1), new Position(2, 1)); // b6
        game.displayBoard();

        game.makeMove(new Position(7, 5), new Position(4, 2)); // Bc4
        game.displayBoard();

        game.makeMove(new Position(1, 6), new Position(2, 6)); // g6
        game.displayBoard();

        game.makeMove(new Position(3, 7), new Position(1, 5)); // Qxf7 Checkmate!
        game.displayBoard();
    }
}
```

---

## Problem 7: Hotel Booking System

### Requirements
- Multiple hotels with rooms
- Room types and pricing
- Reservation management
- Search and availability
- Check-in/Check-out

### Code Implementation

```java
// ============ ENUMS ============
public enum RoomType {
    SINGLE(1, 100),
    DOUBLE(2, 150),
    DELUXE(2, 250),
    SUITE(4, 400);

    private final int maxOccupancy;
    private final double basePrice;

    RoomType(int maxOccupancy, double basePrice) {
        this.maxOccupancy = maxOccupancy;
        this.basePrice = basePrice;
    }

    public int getMaxOccupancy() { return maxOccupancy; }
    public double getBasePrice() { return basePrice; }
}

public enum RoomStatus {
    AVAILABLE, RESERVED, OCCUPIED, MAINTENANCE
}

public enum ReservationStatus {
    CONFIRMED, CHECKED_IN, CHECKED_OUT, CANCELLED
}

// ============ ROOM ============
public class Room {
    private final String id;
    private final String roomNumber;
    private final RoomType type;
    private final int floor;
    private RoomStatus status;
    private final List<String> amenities;
    private double pricePerNight;

    public Room(String id, String roomNumber, RoomType type, int floor) {
        this.id = id;
        this.roomNumber = roomNumber;
        this.type = type;
        this.floor = floor;
        this.status = RoomStatus.AVAILABLE;
        this.amenities = new ArrayList<>();
        this.pricePerNight = type.getBasePrice();
    }

    public void addAmenity(String amenity) {
        amenities.add(amenity);
    }

    public String getId() { return id; }
    public String getRoomNumber() { return roomNumber; }
    public RoomType getType() { return type; }
    public int getFloor() { return floor; }
    public RoomStatus getStatus() { return status; }
    public void setStatus(RoomStatus status) { this.status = status; }
    public double getPricePerNight() { return pricePerNight; }
    public void setPricePerNight(double price) { this.pricePerNight = price; }
    public List<String> getAmenities() { return Collections.unmodifiableList(amenities); }
}

// ============ GUEST ============
public class Guest {
    private final String id;
    private final String name;
    private final String email;
    private final String phone;
    private final String idProof;

    public Guest(String id, String name, String email, String phone, String idProof) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.phone = phone;
        this.idProof = idProof;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }
    public String getPhone() { return phone; }
}

// ============ RESERVATION ============
public class Reservation {
    private final String id;
    private final Guest guest;
    private final Room room;
    private final LocalDate checkInDate;
    private final LocalDate checkOutDate;
    private ReservationStatus status;
    private final double totalAmount;
    private final LocalDateTime createdAt;
    private LocalDateTime actualCheckIn;
    private LocalDateTime actualCheckOut;

    public Reservation(String id, Guest guest, Room room,
                      LocalDate checkInDate, LocalDate checkOutDate) {
        this.id = id;
        this.guest = guest;
        this.room = room;
        this.checkInDate = checkInDate;
        this.checkOutDate = checkOutDate;
        this.status = ReservationStatus.CONFIRMED;
        this.createdAt = LocalDateTime.now();

        long nights = ChronoUnit.DAYS.between(checkInDate, checkOutDate);
        this.totalAmount = room.getPricePerNight() * nights;
    }

    public void checkIn() {
        this.status = ReservationStatus.CHECKED_IN;
        this.actualCheckIn = LocalDateTime.now();
        this.room.setStatus(RoomStatus.OCCUPIED);
    }

    public void checkOut() {
        this.status = ReservationStatus.CHECKED_OUT;
        this.actualCheckOut = LocalDateTime.now();
        this.room.setStatus(RoomStatus.AVAILABLE);
    }

    public void cancel() {
        this.status = ReservationStatus.CANCELLED;
        this.room.setStatus(RoomStatus.AVAILABLE);
    }

    public String getId() { return id; }
    public Guest getGuest() { return guest; }
    public Room getRoom() { return room; }
    public LocalDate getCheckInDate() { return checkInDate; }
    public LocalDate getCheckOutDate() { return checkOutDate; }
    public ReservationStatus getStatus() { return status; }
    public double getTotalAmount() { return totalAmount; }
}

// ============ HOTEL ============
public class Hotel {
    private final String id;
    private final String name;
    private final String address;
    private final String city;
    private final List<Room> rooms;
    private final Map<String, List<Reservation>> roomReservations;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public Hotel(String id, String name, String address, String city) {
        this.id = id;
        this.name = name;
        this.address = address;
        this.city = city;
        this.rooms = new ArrayList<>();
        this.roomReservations = new ConcurrentHashMap<>();
    }

    public void addRoom(Room room) {
        rooms.add(room);
        roomReservations.put(room.getId(), new ArrayList<>());
    }

    public List<Room> findAvailableRooms(LocalDate checkIn, LocalDate checkOut,
                                         RoomType type) {
        lock.readLock().lock();
        try {
            return rooms.stream()
                .filter(r -> r.getType() == type || type == null)
                .filter(r -> isRoomAvailable(r, checkIn, checkOut))
                .collect(Collectors.toList());
        } finally {
            lock.readLock().unlock();
        }
    }

    private boolean isRoomAvailable(Room room, LocalDate checkIn, LocalDate checkOut) {
        if (room.getStatus() == RoomStatus.MAINTENANCE) {
            return false;
        }

        List<Reservation> reservations = roomReservations.get(room.getId());
        return reservations.stream()
            .filter(r -> r.getStatus() == ReservationStatus.CONFIRMED ||
                        r.getStatus() == ReservationStatus.CHECKED_IN)
            .noneMatch(r -> datesOverlap(checkIn, checkOut,
                                        r.getCheckInDate(), r.getCheckOutDate()));
    }

    private boolean datesOverlap(LocalDate start1, LocalDate end1,
                                LocalDate start2, LocalDate end2) {
        return !start1.isAfter(end2.minusDays(1)) &&
               !end1.minusDays(1).isBefore(start2);
    }

    public void addReservation(Reservation reservation) {
        lock.writeLock().lock();
        try {
            roomReservations.get(reservation.getRoom().getId()).add(reservation);
        } finally {
            lock.writeLock().unlock();
        }
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getCity() { return city; }
    public List<Room> getRooms() { return Collections.unmodifiableList(rooms); }
}

// ============ HOTEL BOOKING SERVICE ============
public class HotelBookingService {
    private static HotelBookingService instance;

    private final Map<String, Hotel> hotels = new ConcurrentHashMap<>();
    private final Map<String, Reservation> reservations = new ConcurrentHashMap<>();
    private final Map<String, Guest> guests = new ConcurrentHashMap<>();
    private final AtomicLong idCounter = new AtomicLong(0);

    private HotelBookingService() {}

    public static synchronized HotelBookingService getInstance() {
        if (instance == null) {
            instance = new HotelBookingService();
        }
        return instance;
    }

    public Hotel addHotel(String name, String address, String city) {
        String id = "H" + idCounter.incrementAndGet();
        Hotel hotel = new Hotel(id, name, address, city);
        hotels.put(id, hotel);
        return hotel;
    }

    public Guest registerGuest(String name, String email, String phone, String idProof) {
        String id = "G" + idCounter.incrementAndGet();
        Guest guest = new Guest(id, name, email, phone, idProof);
        guests.put(id, guest);
        return guest;
    }

    public List<Hotel> searchHotels(String city) {
        return hotels.values().stream()
            .filter(h -> h.getCity().equalsIgnoreCase(city))
            .collect(Collectors.toList());
    }

    public List<Room> searchAvailableRooms(String hotelId, LocalDate checkIn,
                                           LocalDate checkOut, RoomType type) {
        Hotel hotel = hotels.get(hotelId);
        if (hotel == null) return Collections.emptyList();

        return hotel.findAvailableRooms(checkIn, checkOut, type);
    }

    public Optional<Reservation> makeReservation(String guestId, String hotelId,
                                                  String roomId, LocalDate checkIn,
                                                  LocalDate checkOut) {
        Guest guest = guests.get(guestId);
        Hotel hotel = hotels.get(hotelId);

        if (guest == null || hotel == null) {
            return Optional.empty();
        }

        Room room = hotel.getRooms().stream()
            .filter(r -> r.getId().equals(roomId))
            .findFirst()
            .orElse(null);

        if (room == null) {
            return Optional.empty();
        }

        // Check availability
        List<Room> available = hotel.findAvailableRooms(checkIn, checkOut, room.getType());
        if (!available.contains(room)) {
            System.out.println("Room not available for selected dates");
            return Optional.empty();
        }

        String reservationId = "R" + idCounter.incrementAndGet();
        Reservation reservation = new Reservation(reservationId, guest, room,
                                                   checkIn, checkOut);

        reservations.put(reservationId, reservation);
        hotel.addReservation(reservation);

        System.out.println("Reservation confirmed: " + reservationId);
        System.out.println("Total: $" + reservation.getTotalAmount());

        return Optional.of(reservation);
    }

    public boolean checkIn(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null ||
            reservation.getStatus() != ReservationStatus.CONFIRMED) {
            return false;
        }

        reservation.checkIn();
        System.out.println("Checked in: " + reservation.getGuest().getName() +
                          " to Room " + reservation.getRoom().getRoomNumber());
        return true;
    }

    public boolean checkOut(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null ||
            reservation.getStatus() != ReservationStatus.CHECKED_IN) {
            return false;
        }

        reservation.checkOut();
        System.out.println("Checked out: " + reservation.getGuest().getName() +
                          " from Room " + reservation.getRoom().getRoomNumber());
        System.out.println("Total bill: $" + reservation.getTotalAmount());
        return true;
    }

    public boolean cancelReservation(String reservationId) {
        Reservation reservation = reservations.get(reservationId);
        if (reservation == null ||
            reservation.getStatus() != ReservationStatus.CONFIRMED) {
            return false;
        }

        reservation.cancel();
        System.out.println("Reservation cancelled: " + reservationId);
        return true;
    }
}

// ============ DEMO ============
public class HotelBookingDemo {
    public static void main(String[] args) {
        HotelBookingService service = HotelBookingService.getInstance();

        // Add hotel and rooms
        Hotel hotel = service.addHotel("Grand Plaza", "123 Main St", "New York");

        for (int i = 1; i <= 5; i++) {
            Room room = new Room("RM" + i, "10" + i, RoomType.SINGLE, 1);
            hotel.addRoom(room);
        }
        for (int i = 6; i <= 10; i++) {
            Room room = new Room("RM" + i, "20" + (i-5), RoomType.DELUXE, 2);
            hotel.addRoom(room);
        }

        // Register guest
        Guest guest = service.registerGuest("John Doe", "john@email.com",
                                           "1234567890", "DL123456");

        // Search and book
        LocalDate checkIn = LocalDate.now().plusDays(1);
        LocalDate checkOut = LocalDate.now().plusDays(3);

        List<Room> available = service.searchAvailableRooms(
            hotel.getId(), checkIn, checkOut, RoomType.DELUXE);

        System.out.println("Available DELUXE rooms: " + available.size());

        if (!available.isEmpty()) {
            Optional<Reservation> reservation = service.makeReservation(
                guest.getId(), hotel.getId(), available.get(0).getId(),
                checkIn, checkOut);

            reservation.ifPresent(r -> {
                // Check in
                service.checkIn(r.getId());

                // Later... check out
                service.checkOut(r.getId());
            });
        }
    }
}
```

---

## Problem 8: LRU Cache

### Requirements
- Fixed capacity cache
- O(1) get and put operations
- Evict least recently used item when full
- Thread-safe implementation

### Code Implementation

```java
// ============ NODE ============
class Node<K, V> {
    K key;
    V value;
    Node<K, V> prev;
    Node<K, V> next;

    public Node(K key, V value) {
        this.key = key;
        this.value = value;
    }
}

// ============ LRU CACHE ============
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> cache;
    private final Node<K, V> head; // Most recently used
    private final Node<K, V> tail; // Least recently used
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();

        // Dummy head and tail
        head = new Node<>(null, null);
        tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }

    public V get(K key) {
        lock.writeLock().lock();
        try {
            Node<K, V> node = cache.get(key);
            if (node == null) {
                return null;
            }

            // Move to front (most recently used)
            moveToHead(node);
            return node.value;
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void put(K key, V value) {
        lock.writeLock().lock();
        try {
            Node<K, V> existing = cache.get(key);

            if (existing != null) {
                // Update existing
                existing.value = value;
                moveToHead(existing);
            } else {
                // Create new node
                Node<K, V> newNode = new Node<>(key, value);

                // Evict if at capacity
                if (cache.size() >= capacity) {
                    Node<K, V> lru = removeTail();
                    cache.remove(lru.key);
                    System.out.println("Evicted: " + lru.key);
                }

                // Add to cache
                cache.put(key, newNode);
                addToHead(newNode);
            }
        } finally {
            lock.writeLock().unlock();
        }
    }

    public void remove(K key) {
        lock.writeLock().lock();
        try {
            Node<K, V> node = cache.remove(key);
            if (node != null) {
                removeNode(node);
            }
        } finally {
            lock.writeLock().unlock();
        }
    }

    private void addToHead(Node<K, V> node) {
        node.prev = head;
        node.next = head.next;
        head.next.prev = node;
        head.next = node;
    }

    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }

    private Node<K, V> removeTail() {
        Node<K, V> lru = tail.prev;
        removeNode(lru);
        return lru;
    }

    public int size() {
        lock.readLock().lock();
        try {
            return cache.size();
        } finally {
            lock.readLock().unlock();
        }
    }

    public void display() {
        lock.readLock().lock();
        try {
            System.out.print("Cache [MRU -> LRU]: ");
            Node<K, V> current = head.next;
            while (current != tail) {
                System.out.print(current.key + "=" + current.value);
                current = current.next;
                if (current != tail) System.out.print(" -> ");
            }
            System.out.println();
        } finally {
            lock.readLock().unlock();
        }
    }
}

// ============ GENERIC CACHE INTERFACE ============
public interface Cache<K, V> {
    V get(K key);
    void put(K key, V value);
    void remove(K key);
    int size();
    void clear();
}

// ============ LRU CACHE WITH EXPIRY ============
public class LRUCacheWithExpiry<K, V> {
    private final int capacity;
    private final long ttlMillis;
    private final Map<K, CacheEntry<V>> cache;
    private final Deque<K> accessOrder;
    private final ReentrantLock lock = new ReentrantLock();
    private final ScheduledExecutorService cleaner;

    static class CacheEntry<V> {
        V value;
        long expiryTime;

        CacheEntry(V value, long ttlMillis) {
            this.value = value;
            this.expiryTime = System.currentTimeMillis() + ttlMillis;
        }

        boolean isExpired() {
            return System.currentTimeMillis() > expiryTime;
        }
    }

    public LRUCacheWithExpiry(int capacity, long ttlMillis) {
        this.capacity = capacity;
        this.ttlMillis = ttlMillis;
        this.cache = new HashMap<>();
        this.accessOrder = new LinkedList<>();
        this.cleaner = Executors.newSingleThreadScheduledExecutor();

        // Periodic cleanup of expired entries
        cleaner.scheduleAtFixedRate(this::removeExpiredEntries,
            ttlMillis, ttlMillis, TimeUnit.MILLISECONDS);
    }

    public V get(K key) {
        lock.lock();
        try {
            CacheEntry<V> entry = cache.get(key);
            if (entry == null) {
                return null;
            }

            if (entry.isExpired()) {
                cache.remove(key);
                accessOrder.remove(key);
                return null;
            }

            // Update access order
            accessOrder.remove(key);
            accessOrder.addFirst(key);

            return entry.value;
        } finally {
            lock.unlock();
        }
    }

    public void put(K key, V value) {
        lock.lock();
        try {
            if (cache.containsKey(key)) {
                cache.get(key).value = value;
                cache.get(key).expiryTime = System.currentTimeMillis() + ttlMillis;
                accessOrder.remove(key);
                accessOrder.addFirst(key);
            } else {
                if (cache.size() >= capacity) {
                    K lru = accessOrder.removeLast();
                    cache.remove(lru);
                    System.out.println("Evicted: " + lru);
                }

                cache.put(key, new CacheEntry<>(value, ttlMillis));
                accessOrder.addFirst(key);
            }
        } finally {
            lock.unlock();
        }
    }

    private void removeExpiredEntries() {
        lock.lock();
        try {
            cache.entrySet().removeIf(entry -> {
                if (entry.getValue().isExpired()) {
                    accessOrder.remove(entry.getKey());
                    System.out.println("Expired: " + entry.getKey());
                    return true;
                }
                return false;
            });
        } finally {
            lock.unlock();
        }
    }

    public void shutdown() {
        cleaner.shutdown();
    }
}

// ============ DEMO ============
public class LRUCacheDemo {
    public static void main(String[] args) {
        System.out.println("=== Basic LRU Cache ===\n");

        LRUCache<String, Integer> cache = new LRUCache<>(3);

        cache.put("A", 1);
        cache.put("B", 2);
        cache.put("C", 3);
        cache.display();

        System.out.println("Get A: " + cache.get("A"));
        cache.display(); // A moves to front

        cache.put("D", 4); // Evicts B (least recently used)
        cache.display();

        cache.put("E", 5); // Evicts C
        cache.display();

        System.out.println("Get B: " + cache.get("B")); // null (evicted)

        System.out.println("\n=== LRU Cache with Expiry ===\n");

        LRUCacheWithExpiry<String, String> expiryCache =
            new LRUCacheWithExpiry<>(3, 2000); // 2 second TTL

        expiryCache.put("X", "value1");
        expiryCache.put("Y", "value2");

        System.out.println("Get X: " + expiryCache.get("X"));

        try {
            Thread.sleep(2500); // Wait for expiry
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        System.out.println("Get X after expiry: " + expiryCache.get("X")); // null

        expiryCache.shutdown();
    }
}
```

---

*Continued in Part 3...*
