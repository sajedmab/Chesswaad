<!DOCTYPE html>
<html lang="ar">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>لعبة الشطرنج المتقدمة</title>
    <style>
        body { text-align: center; font-family: Arial, sans-serif; }
        #status { margin-top: 10px; font-size: 30px; font-weight: bold; }
        .pink-turn { color: pink; }
        .blue-turn { color: skyblue; }
        #board-container { display: flex; justify-content: center; margin-top: 20px; }
        #board {
            display: grid;
            grid-template-columns: repeat(8, 12vw);
            grid-template-rows: repeat(8, 12vw);
            border: 3px solid black;
            width: 96vw;
            max-width: 480px;
        }
        .square {
            width: 100%; height: 100%;
            display: flex; align-items: center; justify-content: center;
            font-size: 5vw; font-weight: bold;
            cursor: pointer;
        }
        .light { background-color: #FFDDDD; }
        .dark { background-color: #DDFFFF; }
        .highlight { background-color: green !important; }
        .check { background-color: red !important; }
        #footer { font-size: 15px; color: gray; position: fixed; bottom: 60px; width: 100%; text-align: center; }
        .popup {
            display: none;
            position: fixed;
            left: 50%;
            top: 50%;
            transform: translate(-50%, -50%);
            background: white;
            padding: 20px;
            border: 2px solid black;
            text-align: center;
        }
        .popup button {
            margin-top: 10px;
            padding: 10px;
            font-size: 16px;
            cursor: pointer;
        }
    </style>
</head>
<body>
    <h1 id="status" class="pink-turn">دور وعد</h1>
    <div id="board"></div>
    <div id="footer">طور بواسطة ساجد</div>
    <div id="popup" class="popup"></div>
    <audio id="laughSound" src="laugh.mp3"></audio>

    <script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.12.0/chess.min.js"></script>
    <script>
        const boardElement = document.getElementById("board");
        const statusEl = document.getElementById("status");
        const popup = document.getElementById("popup");
        const laughSound = document.getElementById("laughSound");
        let game = new Chess();
        let selectedSquare = null;
        let promotionMove = null;

        function createBoard() {
            boardElement.innerHTML = "";
            game.board().forEach((row, rowIndex) => {
                row.forEach((square, colIndex) => {
                    const squareElement = document.createElement("div");
                    squareElement.classList.add("square", (rowIndex + colIndex) % 2 === 0 ? "light" : "dark");
                    squareElement.dataset.position = `${String.fromCharCode(97 + colIndex)}${8 - rowIndex}`;
                    if (square) squareElement.innerHTML = pieceSymbol(square);
                    squareElement.addEventListener("click", () => onSquareClick(squareElement));
                    boardElement.appendChild(squareElement);
                });
            });
            updateStatus();
        }

        function pieceSymbol(piece) {
            if (!piece) return "";
            const symbols = { p: "♟", r: "♜", n: "♞", b: "♝", q: "♛", k: "♚", P: "♙", R: "♖", N: "♘", B: "♗", Q: "♕", K: "♔" };
            return `<span style="color: ${piece.color === 'w' ? 'black' : 'white'};">${symbols[piece.type]}</span>`;
        }

        function showPopup(message, buttons) {
            popup.innerHTML = `<p>${message}</p>${buttons}`;
            popup.style.display = "block";
        }

        function restartGame() {
            popup.style.display = "none";
            game = new Chess();
            createBoard();
            updateStatus();
        }

        function findKing(turn) {
            return game.board().flat().find(sq => sq && sq.type === 'k' && sq.color === turn)?.square;
        }

        function highlightMoves(position) {
            document.querySelectorAll(".square").forEach(el => el.classList.remove("highlight"));
            const moves = game.moves({ square: position, verbose: true });
            moves.forEach(move => {
                const moveSquare = document.querySelector(`[data-position='${move.to}']`);
                if (moveSquare) moveSquare.classList.add("highlight");
            });
        }

        function botMove() {
            const moves = game.moves({ verbose: true });
            if (moves.length === 0) return;
            moves.sort((a, b) => (b.captured ? 1 : 0) - (a.captured ? 1 : 0));
            const bestMove = moves[0];
            game.move(bestMove);
            updateStatus();
            createBoard();
        }

        function onSquareClick(squareElement) {
            const position = squareElement.dataset.position;
            if (!position) return;

            if (selectedSquare) {
                const move = game.move({ from: selectedSquare, to: position, promotion: 'q' });
                if (move) {
                    if (move.flags.includes('c')) laughSound.play();
                    if (move.flags.includes('p')) {
                        promotionMove = { from: selectedSquare, to: position };
                        showPromotionPopup();
                    } else {
                        updateStatus();
                        createBoard();
                        if (game.turn() === 'b') setTimeout(botMove, 500);
                    }
                }
                selectedSquare = null;
                document.querySelectorAll(".square").forEach(el => el.classList.remove("highlight"));
            } else {
                const piece = game.get(position);
                if (piece && piece.color === 'w') {
                    selectedSquare = position;
                    highlightMoves(position);
                }
            }
        }

        function showPromotionPopup() {
            let buttons = '';
            ['q', 'r', 'n', 'b'].forEach(piece => {
                buttons += `<button onclick="promote('${piece}')">${pieceSymbol({ type: piece, color: game.turn() })}</button>`;
            });
            showPopup("اختر الترقية:", buttons);
        }

        function promote(piece) {
            game.move({ from: promotionMove.from, to: promotionMove.to, promotion: piece });
            promotionMove = null;
            popup.style.display = "none";
            updateStatus();
            createBoard();
            if (game.turn() === 'b') setTimeout(botMove, 500);
        }

        function updateStatus() {
            document.querySelectorAll(".square").forEach(el => el.classList.remove("check"));
            if (game.in_check()) {
                const kingSquare = findKing(game.turn());
                if (kingSquare) {
                    const kingElement = document.querySelector(`[data-position='${kingSquare}']`);
                    if (kingElement) kingElement.classList.add("check");
                }
            }

            if (game.in_checkmate()) {
                showPopup(game.turn() === 'w' ? "ياسمين تفوز!" : "وعد تفوز!", '<button onclick="restartGame()">إعادة اللعب</button>');
            } else if (game.in_stalemate()) {
                showPopup("انتهت اللعبة بالتعادل!", '<button onclick="restartGame()">إعادة اللعب</button>');
            } else {
                statusEl.innerText = game.turn() === 'w' ? "دور وعد" : "دور ياسمين";
                statusEl.classList.toggle("pink-turn", game.turn() === 'w');
                statusEl.classList.toggle("blue-turn", game.turn() === 'b');
            }
        }

        createBoard();
    </script>
</body>
</html>
