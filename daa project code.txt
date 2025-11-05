import tkinter as tk
import random
import time
from queue import Queue, PriorityQueue

CELL_SIZE = 25
DELAY = 0.01
ROWS, COLS = 20, 30  # Large maze

class MazeApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Random Solvable Maze - BFS, DFS, A Comparison")

        self.canvas = tk.Canvas(root, width=COLS*CELL_SIZE, height=ROWS*CELL_SIZE)
        self.canvas.pack()

        self.buttons_frame = tk.Frame(root)
        self.buttons_frame.pack(pady=5)
        tk.Button(self.buttons_frame, text="Generate Maze", command=self.generate_maze).pack(side="left", padx=5)
        tk.Button(self.buttons_frame, text="BFS", command=self.run_bfs).pack(side="left", padx=5)
        tk.Button(self.buttons_frame, text="DFS", command=self.run_dfs).pack(side="left", padx=5)
        tk.Button(self.buttons_frame, text="A", command=self.run_a).pack(side="left", padx=5)
        tk.Button(self.buttons_frame, text="Compare Searches", command=self.compare_searches).pack(side="left", padx=5)
        tk.Button(self.buttons_frame, text="Reset Maze", command=self.draw_maze).pack(side="left", padx=5)

        self.info_label = tk.Label(root, text="", justify="left", font=("Arial",12))
        self.info_label.pack(pady=5)

        self.search_results = {}
        self.generate_maze()

    # ---------------- Maze Generation ----------------
    def generate_maze(self):
        self.maze_map = [['#' for _ in range(COLS)] for _ in range(ROWS)]
        self.start = (1,1)
        self.goal = (ROWS-2, COLS-2)
        self.maze_map[self.start[0]][self.start[1]] = 'S'
        self.maze_map[self.goal[0]][self.goal[1]] = 'G'

        # Carve guaranteed path
        x, y = self.start
        while (x,y) != self.goal:
            options = []
            if x < self.goal[0]: options.append((x+1, y))
            if y < self.goal[1]: options.append((x, y+1))
            if x > self.goal[0]: options.append((x-1, y))
            if y > self.goal[1]: options.append((x, y-1))
            x, y = random.choice(options)
            if self.maze_map[x][y] not in ['S','G']:
                self.maze_map[x][y] = '.'

        # Random empty spaces to create multiple paths
        for i in range(1, ROWS-1):
            for j in range(1, COLS-1):
                if self.maze_map[i][j] == '#':
                    if random.random() < 0.55:  # Increased empty space probability
                        self.maze_map[i][j] = '.'

        self.draw_maze()
        self.search_results.clear()

    # ---------------- Drawing Maze ----------------
    def draw_maze(self):
        self.grid = []
        for i in range(ROWS):
            line = []
            for j in range(COLS):
                ch = self.maze_map[i][j]
                color = 'black' if ch=='#' else 'white'
                if ch=='S': color='green'
                if ch=='G': color='red'
                rect = self.canvas.create_rectangle(j*CELL_SIZE, i*CELL_SIZE,
                                                    (j+1)*CELL_SIZE, (i+1)*CELL_SIZE,
                                                    fill=color, outline='gray')
                line.append(rect)
            self.grid.append(line)
        self.root.update()

    def color_cell(self, pos, color):
        i,j = pos
        self.canvas.itemconfig(self.grid[i][j], fill=color)
        self.root.update()
        time.sleep(DELAY)

    def get_neighbors(self, pos):
        x,y = pos
        for dx,dy in [(0,1),(1,0),(0,-1),(-1,0)]:
            nx,ny = x+dx, y+dy
            if 0<=nx<ROWS and 0<=ny<COLS and self.maze_map[nx][ny]!='#':
                yield (nx,ny)

    # ---------------- Search Runner ----------------
    def run_search(self, search_func, name):
        start_time = time.time()
        path, visited = search_func(self.start, self.goal)
        elapsed = round(time.time() - start_time, 4)

        self.search_results[name] = {
            "path": path,
            "visited": visited,
            "length": len(path) if path else float('inf'),
            "time": elapsed
        }

        self.draw_maze()

        for cell in visited:
            if self.maze_map[cell[0]][cell[1]] not in ['S','G']:
                self.color_cell(cell,'light blue')

        if path:
            for cell in path:
                if self.maze_map[cell[0]][cell[1]] not in ['S','G']:
                    self.color_cell(cell,'yellow')
            self.info_label.config(text=f"{name} → Path length: {len(path)}, Time: {elapsed}s")
        else:
            self.info_label.config(text=f"{name} → No path found!")

    # ---------------- BFS (randomized neighbors) ----------------
    def bfs(self, start, goal):
        q = Queue()
        q.put((start,[start]))
        visited = set()
        path_found = None
        while not q.empty():
            curr, path = q.get()
            if curr in visited: continue
            visited.add(curr)
            if curr == goal:
                path_found = path
                break
            neighbors = list(self.get_neighbors(curr))
            random.shuffle(neighbors)  # Randomize BFS
            for nxt in neighbors:
                if nxt not in visited:
                    q.put((nxt,path+[nxt]))
        return path_found, visited

    # ---------------- DFS (randomized neighbors) ----------------
    def dfs(self, start, goal):
        stack = [(start,[start])]
        visited = set()
        path_found = None
        while stack:
            curr, path = stack.pop()
            if curr in visited: continue
            visited.add(curr)
            if curr == goal:
                path_found = path
                break
            neighbors = list(self.get_neighbors(curr))
            random.shuffle(neighbors)  # Randomize DFS
            for nxt in neighbors:
                if nxt not in visited:
                    stack.append((nxt,path+[nxt]))
        return path_found, visited

    # ---------------- A Search (randomized tie-break) ----------------
    def a(self, start, goal):
        pq = PriorityQueue()
        pq.put((0,random.random(), start,[start]))  # random tie-breaker
        visited = set()
        path_found = None
        while not pq.empty():
            f_cost, _, curr, path = pq.get()
            if curr in visited: continue
            visited.add(curr)
            if curr == goal:
                path_found = path
                break
            neighbors = list(self.get_neighbors(curr))
            random.shuffle(neighbors)  # Randomize A
            for nxt in neighbors:
                g = len(path)
                h = abs(nxt[0]-goal[0]) + abs(nxt[1]-goal[1])
                pq.put((g+h, random.random(), nxt, path+[nxt]))
        return path_found, visited

    # ---------------- Button commands ----------------
    def run_bfs(self): self.run_search(self.bfs,"BFS")
    def run_dfs(self): self.run_search(self.dfs,"DFS")
    def run_a(self): self.run_search(self.a,"A")

    # ---------------- Compare Searches ----------------
    def compare_searches(self):
        for s in ["BFS","DFS","A"]:
            if s not in self.search_results:
                self.run_search(getattr(self, s.lower()), s)

        result_text = "Comparison of Searches:\n"
        for k,v in self.search_results.items():
            result_text += f"{k}: Length={v['length']}, Time={v['time']}s\n"

        # Best shortest path
        best_path = min(self.search_results.items(), key=lambda x: x[1]['length'])[0]
        # Fastest search
        fastest = min(self.search_results.items(), key=lambda x: x[1]['time'])[0]

        result_text += f"\nBest (shortest path): {best_path}\nFastest search: {fastest}"
        self.info_label.config(text=result_text)

# ---------------- Main ----------------
if __name__ == "__main__":
    root = tk.Tk()
    app = MazeApp(root)
    root.mainloop()