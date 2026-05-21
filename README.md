# index.htlm
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Chess Mobile Pro</title>
    <!-- Chargement des bibliothèques nécessaires : logique du jeu et styles de l'échiquier -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.10.3/chess.min.js"></script>
    <link rel="stylesheet" href="https://unpkg.com/@chrisoakman/chessboardjs@1.0.0/dist/chessboard-1.0.0.min.css">
    <script src="https://code.jquery.com/jquery-3.5.1.min.js"></script>
    <script src="https://unpkg.com/@chrisoakman/chessboardjs@1.0.0/dist/chessboard-1.0.0.min.js"></script>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
            background-color: #161512;
            color: #fff;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            margin: 0;
            padding: 10px;
            box-sizing: border-box;
        }
        h1 {
            margin: 5px 0;
            font-size: 1.5rem;
            color: #f0d9b5;
        }
        #status {
            margin-bottom: 15px;
            font-size: 1rem;
            font-weight: bold;
            color: #b58863;
            text-align: center;
        }
        #board {
            width: 100%;
            max-width: 360px; /* Taille idéale pour les écrans de smartphone */
            box-shadow: 0 5px 15px rgba(0,0,0,0.5);
        }
        .btn-restart {
            margin-top: 20px;
            padding: 12px 24px;
            font-size: 1rem;
            background-color: #b58863;
            color: #fff;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            -webkit-tap-highlight-color: transparent;
        }
        .btn-restart:active {
            background-color: #f0d9b5;
            color: #161512;
        }
    </style>
</head>
<body>

    <h1>Chess Mobile</h1>
    <div id="status">À vous de jouer (Blancs)</div>
    
    <!-- L'échiquier s'adapte automatiquement à la largeur du téléphone -->
    <div id="board"></div>

    <button class="btn-restart" onclick="recommencer()">Nouvelle Partie</button>

    <script>
        let board = null;
        const game = new Chess();
        const $status = $('#status');
        let originSquare = null;

        // IA Basique pour le jeu sur mobile (Minimax simplifié à 2 coups d'avance)
        function evaluateBoard(boardState) {
            let totalEvaluation = 0;
            for (let i = 0; i < 8; i++) {
                for (let j = 0; j < 8; j++) {
                    const piece = boardState[i][j];
                    if (piece) {
                        const value = getPieceValue(piece.type);
                        totalEvaluation += (piece.color === 'w') ? -value : value;
                    }
                }
            }
            return totalEvaluation;
        }

        function getPieceValue(type) {
            if (type === 'p') return 10;
            if (type === 'n') return 30;
            if (type === 'b') return 30;
            if (type === 'r') return 50;
            if (type === 'q') return 90;
            if (type === 'k') return 900;
            return 0;
        }

        function makeAIMove() {
            const moves = game.moves({ verbose: true });
            if (moves.length === 0) return;

            // Tri basique pour privilégier les captures
            moves.sort((a, b) => (b.captured ? 1 : 0) - (a.captured ? 1 : 0));
            
            // Sélection du meilleur coup disponible immédiatement
            let bestMove = moves[0];
            let bestValue = -9999;

            for (let i = 0; i < moves.length; i++) {
                game.move(moves[i]);
                let boardValue = evaluateBoard(game.board());
                game.undo();
                if (boardValue > bestValue) {
                    bestValue = boardValue;
                    bestMove = moves[i];
                }
            }

            game.move(bestMove);
            board.position(game.fen());
            updateStatus();
        }

        // Gestion tactile par clics successifs (Case A puis Case B)
        function onSquareClick(square) {
            // Si c'est le tour de l'ordinateur, on bloque les entrées
            if (game.turn() === 'b') return;

            if (originSquare === null) {
                // Premier clic : sélection de la pièce
                const piece = game.get(square);
                if (piece && piece.color === game.turn()) {
                    originSquare = square;
                    // Met en valeur visuellement la case sélectionnée
                    $('#board .square-' + square).css('background', '#b58863');
                }
            } else {
                // Deuxième clic : tentative de déplacement
                const move = game.move({
                    from: originSquare,
                    to: square,
                    promotion: 'q' // Promotion automatique en Reine pour le mobile
                });

                // Réinitialise la couleur de fond des cases
                board.clearMoveHighlight();
                originSquare = null;

                if (move === null) {
                    // Si le coup est invalide, on tente de sélectionner cette nouvelle case si elle contient une pièce à soi
                    const piece = game.get(square);
                    if (piece && piece.color === game.turn()) {
                        originSquare = square;
                        $('#board .square-' + square).css('background', '#b58863');
                    }
                    return;
                }

                // Coup joueur valide, mise à jour du plateau
                board.position(game.fen());
                updateStatus();

                // L'IA joue après un court délai pour plus de réalisme
                if (!game.game_over()) {
                    $status.html('L\'ordinateur réfléchit...');
                    setTimeout(makeAIMove, 250);
                }
            }
        }

        function updateStatus() {
            let statusText = '';
            let moveColor = (game.turn() === 'w') ? 'Blancs' : 'Noirs';

            if (game.in_checkmate()) {
                statusText = 'Fin de partie. ' + (game.turn() === 'w' ? 'Les Noirs gagnent !' : 'Les Blancs gagnent !');
            } else if (game.in_draw()) {
                statusText = 'Match nul !';
            } else {
                statusText = 'Tour des ' + moveColor;
                if (game.in_check()) {
                    statusText += ' (Échec !)';
                }
            }
            $status.html(statusText);
        }

        const config = {
            draggable: false, // Désactivé car le glisser-déposer échoue souvent sur navigateur mobile
            position: 'start',
            pieceTheme: 'https://unpkg.com/@chrisoakman/chessboardjs@1.0.0/dist/img/chesspieces/wikipedia/{piece}.png'
        };
        
        board = Chessboard('board', config);

        // Liaison de l'événement de clic tactile sur les cases
        $('#board').on('click', '.square-55d63', function() {
            const square = $(this).attr('data-square');
            onSquareClick(square);
        });

        // Nettoyage des surbrillances personnalisées
        board.clearMoveHighlight = function() {
            $('#board .square-55d63').css('background', '');
        };

        function recommencer() {
            game.clear();
            game.load('rnbqkbnr/pppppppp/8/8/8/8/PPPPPPPP/RNBQKBNR w KQkq - 0 1');
            board.position('start');
            originSquare = null;
            board.clearMoveHighlight();
            $status.html('À vous de jouer (Blancs)');
        }
    </script>
</body>
</html>
