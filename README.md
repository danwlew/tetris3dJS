--

# Dokumentacja 3D Tetris

**Spis treści**  
1. [Wprowadzenie](#wprowadzenie)  
2. [Struktura i główne elementy kodu](#struktura)  
3. [Opis klas i funkcji](#opis-klas)  
   - [Funkcje pomocnicze: `getShapeCoords()`, `updateNextPiecePreview()`](#funkcje-pomocnicze)  
   - [Klasa `Shape3D`](#klasa-shape3d)  
   - [Klasa `Tetris3D`](#klasa-tetris3d)  
   - [Funkcje globalne: `init()`, `animate()`, `onWindowResize()`](#funkcje-globalne)  
4. [Nowe kształty klocków: `3x1`, `2x1`, `N`, `M`](#nowe-kształty)  
5. [Cienie na klockach](#cienie-na-klockach)  
6. [Kontrastowe krawędzie](#kontrastowe-krawędzie)  
7. [Podgląd „Next Piece”](#podgląd-next-piece)  
8. [Wskazówki końcowe](#wskazówki-końcowe)

---

<a name="wprowadzenie"></a>
## 1. Wprowadzenie

Ten projekt to **3D Tetris** napisany w JavaScript z użyciem **Three.js**. Celem jest stworzenie trójwymiarowej wersji klasycznej gry Tetris, z dodatkowymi elementami:

- **Cienie** (rzeczywiste z użyciem `shadowMap` w Three.js)  
- **Kontrastowe krawędzie** bloczków, by lepiej je było widać  
- **Nowe klocki** (oprócz tradycyjnych „I”, „O”, „L”, „T” – również `3x1`, `2x1`, `N` i `M`)  
- **Podgląd następnego klocka** (tzw. „Next Piece”), renderowany w osobnej mini-scenie

Rozgrywka polega na tym, że kolejne klocki spadają (osią Y w dół) w trójwymiarowej studni o wymiarach 10×30×10. Gracz może:

- **Przesuwać** je w płaszczyźnie XZ (relatywnie do widoku kamery, za pomocą strzałek).  
- **Obracać** w pełnym 3D (wokół osi X, Y lub Z).  
- **Przyspieszać** spadanie (Soft Drop / Hard Drop).  
- **Kasować** pełne warstwy w osi Y.  

Cały kod, który niniejsza dokumentacja opisuje, znajduje się w pliku HTML/JS (lub w module JS). Możesz go uruchomić bezpośrednio w przeglądarce.

---

<a name="struktura"></a>
## 2. Struktura i główne elementy kodu

1. **Deklaracje stałych**:  
   - `BOARD_WIDTH`, `BOARD_HEIGHT`, `BOARD_DEPTH` – wymiary studni.  
   - `FALL_SPEED_BASE` – bazowy czas spadania klocka (ms).  
   - `COLORS` – tablica z dostępnymi kolorami.  
   - `PIECE_TYPES` – tablica z nazwami kształtów.  

2. **Zmienne globalne**:  
   - `scene`, `camera`, `renderer` – główne elementy Three.js.  
   - `tetris` – instancja głównej klasy `Tetris3D`.  
   - `nextScene`, `nextCamera` – osobna scena + kamera do podglądu „Next Piece”.  
   - `nextPieceMesh` – obiekt (Group) przechowujący wizualizację następnego klocka.  

3. **`init()`**: Inicjalizacja sceny głównej. Tworzy `camera`, `renderer`, dodaje światła, włącza cienie.  

4. **`initNextScene()`**: Inicjuje osobną mini-scenę do renderowania kolejnego klocka (podgląd).  

5. **Pętla `animate()`**:  
   - Renderuje scenę główną na całym oknie.  
   - Renderuje też mini-scenę w rogu ekranu (z użyciem `scissorTest`).  

6. **Klasa `Tetris3D`**:  
   - Zarządza tablicą 3D (`this.grid[x][y][z]`).  
   - Tworzy i obsługuje bieżący klocek (`this.currentPiece`).  
   - Generuje klocki w metodach `spawnPiece()` i `generateNextPiece()` (by z góry wiedzieć, co będzie następne).  
   - Obsługuje zdarzenia klawiatury (ruch, obrót, drop).  
   - Gdy klocek osiągnie dno, wywołuje `lockPiece()` i kasuje pełne warstwy (`checkFullLayers()`).  

7. **Klasa `Shape3D`**:  
   - Reprezentuje **pojedynczy klocek** jako `THREE.Group` zawierający sub-bloczki (1×1×1).  
   - Każdy sub-bloczek ma `castShadow = true` i `receiveShadow = true`.  
   - Metody: `setPosition()`, `getGlobalPositions3D()`, `rotateAxis()`.  

8. **Funkcje pomocnicze**  
   - `getShapeCoords(type)` – zwraca tablicę współrzędnych kształtu w układzie (x,z).  
   - `updateNextPiecePreview(type, color)` – rysuje w mini-scenie nadchodzący klocek.  

---

<a name="opis-klas"></a>
## 3. Opis klas i funkcji

Poniżej znajduje się opis najważniejszych funkcji i metod. Każda z nich będzie omówiona w sposób prosty, z zaznaczeniem, **kiedy** i **przez co** jest wywoływana oraz **co robi**.

<a name="funkcje-pomocnicze"></a>
### 3.1. Funkcje pomocnicze: `getShapeCoords()` i `updateNextPiecePreview()`

#### `getShapeCoords(type)`

- **Gdzie się znajduje**: w kodzie globalnym, poza klasami.  
- **Co robi**: Na podstawie nazwy kształtu (np. `'I'`, `'O'`, `'3x1'`) zwraca tablicę współrzędnych sub-bloczków. Każda współrzędna to `{ x, z }`.  
- **Kto wywołuje**:  
  - Klasa `Shape3D`, gdy tworzy nowy klocek w głównej scenie.  
  - Funkcja `updateNextPiecePreview()` przy generowaniu wizualizacji w mini-scenie.  

#### `updateNextPiecePreview(type, color)`

- **Gdzie się znajduje**: w kodzie globalnym.  
- **Co robi**: Tworzy w mini-scenie (`nextScene`) grupę sub-bloczków reprezentujących kształt `type` i kolor `color`. Jeśli istniał poprzedni obiekt klocka, usuwa go z mini-scenie.  
- **Kto wywołuje**: Metoda `Tetris3D.generateNextPiece()` – za każdym razem, gdy w grze generowany jest nowy klocek do podglądu.  

---

<a name="klasa-shape3d"></a>
### 3.2. Klasa `Shape3D`

Reprezentuje **pojedynczy klocek** w głównej scenie.  
- **Konstruktor**: `constructor(type, color)`  
  - Tworzy `this.group = new THREE.Group()` i dodaje do głównej sceny (`scene.add(...)`).  
  - Z `getShapeCoords(type)` pobiera listę współrzędnych i dla każdej z nich tworzy `Mesh` (sześcian 1×1×1) + krawędzie (EdgesGeometry).  
  - Ustawia `mesh.castShadow = mesh.receiveShadow = true` – dzięki temu klocki rzucają cienie wzajemnie na siebie.  
- **`setPosition(x, y, z)`**  
  - Ustawia `this.position` (logiczna pozycja) i **grupę** (`this.group.position`) na te współrzędne.  
- **`getGlobalPositions3D()`**  
  - Dla każdego sub-bloczka oblicza **rzeczywiste** (globalne) współrzędne (x,y,z), uwzględniając `this.group.position` i `this.group.rotation` (z użyciem quaterniona).  
  - Zaokrągla wartości do liczb całkowitych.  
  - Używane jest do sprawdzania kolizji (czy klocek mieści się w siatce i nie nachodzi na inne klocki).  
- **`rotateAxis(axis, clockwise)`**  
  - Obraca klocek wokół wskazanej osi (`'x'`, `'y'`, `'z'`) o 90°.  
  - Zapisuje starą rotację (do ewentualnego cofnięcia, jeśli okaże się, że jest kolizja).  

---

<a name="klasa-tetris3d"></a>
### 3.3. Klasa `Tetris3D`

Główna klasa logiki Tetrisa:

1. **`constructor()`**  
   - Tworzy trójwymiarową tablicę `this.grid[x][y][z]`, przechowującą informację, który klocek (Shape3D) zajmuje daną pozycję (lub `null`).  
   - Wywołuje `initSceneHelpers()` (dodanie podłogi, krawędzi studni).  
   - Podpina nasłuch klawiatury w `initControls()`.  
   - Inicjuje wartości `this.nextType` i `this.nextColor` przez `generateNextPiece()`, po czym tworzy pierwszy klocek przez `spawnPiece()`.  
   - Uruchamia `startFalling()`.  

2. **`initSceneHelpers()`**  
   - Tworzy **podłogę** (plane geometry) o wymiarach 10×10, leżącą na y=0 i obraca ją, by była w płaszczyźnie XZ.  
   - Ustawia `floorMesh.receiveShadow = true`, by cienie klocków padały na podłogę.  
   - Rysuje obrys studni (EdgesGeometry) 10×30×10.  

3. **`initControls()`**  
   - Dodaje `addEventListener('keydown', ...)`.  
   - W zależności od klawisza, wywołuje funkcje:  
     - Ruch w płaszczyźnie XZ: `movePieceRelative(...)`  
     - Soft/Hard drop: `softDrop()`, `hardDrop()`  
     - Obrót klocka: `rotatePiece(axis, clockwise)`  
     - Obrót i reset kamery: `rotateView()`, `resetCameraView()`  

4. **`generateNextPiece()`**  
   - Losuje `this.nextType` z `PIECE_TYPES` i `this.nextColor` z `COLORS`.  
   - Wywołuje `updateNextPiecePreview(...)` – rysuje podgląd w mini-scenie.  

5. **`spawnPiece()`**  
   - Tworzy nowy klocek `Shape3D` z typem i kolorem pobranym z `this.nextType` i `this.nextColor`.  
   - Ustawia go na górze studni, np. x≈4, y=28, z≈4.  
   - Następnie **ponownie** wywołuje `generateNextPiece()`, aby przygotować kolejny klocek w kolejce.  
   - Jeśli zaraz po stworzeniu wystąpi kolizja, wywołuje `resetGame()` (koniec gry).  

6. **`startFalling()`** / **`stopFalling()`**  
   - Ustawia `fallTimer = setInterval(() => this.softDrop(), spd)`, gdzie `spd = FALL_SPEED_BASE / level`.  
   - Zatrzymuje timer przez `clearInterval(fallTimer)`.  

7. **`movePieceRelative(direction, sideways)`**  
   - Oblicza wektor `forward` z kamery (ignorując oś Y) i wektor `right` (prostopadły do forward).  
   - W zależności od klawisza (up/down, left/right) przesuwa klocek o 1 w odpowiednim kierunku.  
   - Sprawdza kolizję przez `canPlace(...)`.  

8. **`softDrop()` / `hardDrop()`**  
   - Odpowiadają za ruch klocka w osi Y (w dół).  
   - `softDrop()` próbuje zejść o 1.  
   - `hardDrop()` schodzi maksymalnie (w pętli).  
   - Kiedy **nie można** zejść niżej, następuje `lockPiece()`.  

9. **`rotatePiece(axis, clockwise)`**  
   - Pyta klocek (`this.currentPiece`) o zmianę rotacji.  
   - Następnie sprawdza kolizje w nowej orientacji.  
   - Jeśli kolizja, cofa rotację.  

10. **`lockPiece()`**  
    - Przerywa spadanie (`stopFalling()`).  
    - Pozycje sub-bloczków (z `getGlobalPositions3D()`) trafiają do `this.grid[x][y][z]`.  
    - Następnie **kasujemy pełne warstwy** (`checkFullLayers()`) i tworzymy nowy klocek (`spawnPiece()`).  

11. **`checkFullLayers()`** / **`isLayerFull(ly)`**  
    - Sprawdza, czy cały poziom (y=ly) w siatce jest zapełniony.  
    - Jeśli tak, wywołuje `clearLayer(ly)`.  

12. **`clearLayer(ly)`**  
    - Usuwa bloczki z warstwy `ly` (usuwa `group` z `scene`).  
    - Przesuwa wszystko powyżej o 1 w dół.  
    - Wywołuje `rebuildScene()`, by przebudować aktualny układ klocków w scenie (technika „brutalna”, ale skuteczna).  

13. **`rebuildScene()`**  
    - Usuwa stare obiekty z `scene`.  
    - Dla każdego klocka (Shape3D) zbiera pozycje w tablicy `shapeMap`, a potem tworzy `Group` z sub-bloczkami.  
    - Ustawia `castShadow` i `receiveShadow` na `true` w nowo tworzonych bloczkach.  

14. **`updateLevel()`**  
    - Gdy gracz zdobywa punkty (kasuje warstwy), level rośnie co 1000 punktów.  
    - Im wyższy level, tym szybsze spadanie (mniejszy `setInterval`).  

15. **Funkcje kamery**:  
    - **`rotateView()`** – przesuwa kamerę w kółko o 90° co naciśnięcie Enter.  
    - **`resetCameraView()`** – ustawia kamerę z powrotem w `CAMERA_DEFAULT_POS`.  

16. **`canPlace(shape, nx, ny, nz)`**  
    - Sprawdza, czy klocek może się znaleźć w pozycji `(nx,ny,nz)` bez kolizji.  
    - Tymczasowo przesuwa klocek w to miejsce, wywołuje `getGlobalPositions3D()`, analizuje, czy wszystkie sub-bloczki są w siatce i czy `this.grid[x][y][z]` jest wolne.  
    - Jeśli jest kolizja, cofa klocek do starej pozycji.  

17. **`isValidCoord(x,y,z)`**  
    - Sprawdza, czy `(x,y,z)` mieści się w granicach studni `10×30×10`.  

---

<a name="funkcje-globalne"></a>
### 3.4. Funkcje globalne: `init()`, `animate()`, `onWindowResize()`

#### `init()`
1. Tworzy **główną scenę**: `scene`.  
2. Ustawia **kamerę perspektywiczną** (pozycja `CAMERA_DEFAULT_POS`, patrzy na środek studni).  
3. Konfiguruje **renderer**: ustawia rozmiar, włącza `shadowMap.enabled = true`.  
4. Dodaje **światło kierunkowe** (`DirectionalLight`) `dirLight.castShadow = true`.  
   - Ustawia parametry kamery cienia (`left, right, top, bottom`).  
5. Dodaje **ambient light** (rozproszone).  
6. Wywołuje `initNextScene()`, aby przygotować mini-scenę do `Next Piece`.  
7. Tworzy instancję `tetris = new Tetris3D()`.  
8. Uruchamia pętlę `animate()`.

#### `animate()`
1. Wywoływana w pętli `requestAnimationFrame`.  
2. Najpierw renderuje scenę główną na całym oknie (`renderer.setViewport(...), render(scene, camera)`).  
3. Następnie, w prawym-górnym rogu (np. 100×100 px) aktywuje `renderer.setScissorTest(true)`, ustawia scissor i viewport na mały kwadrat.  
4. Renderuje mini-scenę `nextScene` z `nextCamera`.  
5. Wyłącza `scissorTest(false)`.  

#### `onWindowResize()`
1. Dostosowuje rozmiar renderera do nowej wielkości okna.  
2. Aktualizuje `camera.aspect` i wywołuje `camera.updateProjectionMatrix()`.  

---

<a name="nowe-kształty"></a>
## 4. Nowe kształty klocków: `3x1`, `2x1`, `N`, `M`

- `3x1`: poziomy klocek z 3 bloczków w linii.  
- `2x1`: poziomy klocek z 2 bloczków.  
- `N`: kształt podobny do litery „S” (2 bloczki w górnym rzędzie, przesunięte o 1 w prawo i 2 bloczki w dolnym rzędzie, przesunięte?).  
- `M`: **6-bloczkowy** kształt:  
  ```
  row=0 => (0,0), (1,0)
  row=1 => (1,1)
  row=2 => (1,2)
  row=3 => (0,3), (1,3)
  ```
  Dzięki temu tworzy wydłużony „zakręcony” klocek.

Definicje współrzędnych znajdują się w funkcji `getShapeCoords(type)`.

---

<a name="cienie-na-klockach"></a>
## 5. Cienie na klockach

Aby cienie były widoczne **na** innych klockach, każdy bloczek w `Shape3D` ma:
```js
mesh.castShadow = true;
mesh.receiveShadow = true;
```
- **`castShadow = true`** sprawia, że bloczek rzuca cień na inne obiekty.  
- **`receiveShadow = true`** pozwala, by cień rzucany przez inne bloczki wyświetlał się na jego powierzchni.  

**Warunek**: musi być włączone mapowanie cieni w `renderer.shadowMap.enabled = true`, a światło musi mieć `light.castShadow = true`. W naszym przypadku używamy `DirectionalLight`. Jego parametry (np. `shadow.camera.left`, `shadow.camera.right`) muszą pokrywać zasięg studni.

---

<a name="kontrastowe-krawędzie"></a>
## 6. Kontrastowe krawędzie

Każdy sub-bloczek (sześcian) ma doklejony `LineSegments` z `EdgesGeometry`. Materiał linii to:
```js
new THREE.LineBasicMaterial({ color: 0x000000 })
```
co daje wyraźny, **czarny** kontur na kolorowym bloczku.

---

<a name="podgląd-next-piece"></a>
## 7. Podgląd „Next Piece”

Podgląd kolejnego klocka realizujemy przez:
- **Drugą scenę (`nextScene`)** i **drugą kamerę (`nextCamera`)**.  
- Funkcję `updateNextPiecePreview(type, color)`, która tworzy w `nextScene` mały zestaw bloczków odpowiadających nadchodzącemu klockowi.  
- W pętli `animate()` używamy `renderer.setScissor(...)` i `renderer.setViewport(...)`, by w prawym-górnym rogu okna (np. 100×100 pikseli) wyrenderować `nextScene` z `nextCamera`.  

`Tetris3D` posiada zmienne `this.nextType` i `this.nextColor`, które się aktualizują za każdym razem, gdy tworzymy nowy klocek do gry. W metodzie `spawnPiece()` bierzemy tamte wartości, a później generujemy nowe przez `generateNextPiece()`.

---

<a name="wskazówki-końcowe"></a>
## 8. Wskazówki końcowe

- **Oświetlenie**: Aby cienie były dobrze widoczne, studnia jest otwarta (tylko krawędzie), a światło kierunkowe jest ustawione wysoko nad planszą.  
- **Kolizje w 3D**: Kod prototypowo zaokrągla globalne pozycje sub-bloczków (`Math.round`). Gdybyś chciał bardziej precyzyjne obroty, trzeba rozbudować logikę pivotów i bounding-boxów.  
- **Kształty** `M` czy `N` są nietypowe i mogą zachowywać się „nie-tettrisowo”, ale jest to część założonego rozszerzenia.  
- **Dalszy rozwój**: Można dodać sterowanie myszą, bardziej zaawansowany obrót kamery (np. orbit controls), czy animowane spadanie.  


---
---  

**Koniec dokumentacji**.  
