[← Оглавление](../../README.md)


Дан неориентированный граф. Найдите длину минимального пути между двумя вершинами.

## Формат ввода

В первой строке записано целое число NN (1≤N≤1001≤N≤100) – количество вершин в графе.

Далее записывается матрица смежности — NN строк, в каждой из которых содержится NN чисел 0 или 1, разделённых пробелом. Число 0 означает отсутствие ребра, а 1 — наличие ребра.

В последней строке задаются номера двух вершин — начальной и конечной.

Вершины нумеруются с единицы.

## Формат вывода

Выведите длину кратчайшего пути — минимальное количество ребер, которые нужно пройти.

Если пути нет, нужно вывести -1.

```python
import sys
from collections import deque

  

def main():
    n = int(input())
    graph = {}

    for vertex in range(1, n + 1):
        buffer = list(map(str, input().split()))
        neighbors = []

        for i in range(len(buffer)):
            if buffer[i] == '1':
                neighbors.append(i + 1)

        graph[vertex] = neighbors

    start, finish = map(int, input().split())

    if start == finish:
        print(0)
        return

  

    distance = {start: 0}
    queue = deque([start])

    # BFS
    while queue:
        current_vertex = queue.popleft()

        for neighbor in graph[current_vertex]:
            if neighbor not in distance:
                distance[neighbor] = distance[current_vertex] + 1

                if neighbor == finish:
                    print(distance[neighbor])
                    return

                queue.append(neighbor)

    print(-1)

if __name__ == '__main__':
    main()
```
