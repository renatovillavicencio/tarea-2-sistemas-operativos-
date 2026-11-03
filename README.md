# Tarea 2: Torneo de Komikan Concurrente
**Sistemas Operativos**
**Integrantes:**
- Tomás Hernández
- Renato Villavicencio

---

## 1. Introducción y Descripción General
En este proyecto desarrollamos la infraestructura completa para el **Torneo Multijugador de Komikan Concurrente en Tiempo Real**. El Komikan es un juego tradicional mapuche asimétrico que enfrenta a un Puma contra 12 Perros en un tablero de 5x5 casillas acoplado a un triángulo (trono).

A diferencia de las versiones por turnos, esta simulación se ejecuta de manera simultánea e independiente en tiempo real: cada ficha es modelada como un hilo o proceso concurrente que intenta reclamar casillas del tablero ejecutando ciclos de decisión con retardos aleatorios en milisegundos. El sistema implementa un servidor central de emparejamiento (Matchmaker), administración de mesas de juego concurrentes mediante árbitros (Jueces) y mecanismos estrictos de sincronización y resiliencia para evitar condiciones de carrera, deadlocks y fugas de recursos en el sistema operativo.

---

## 2. Explicación de Funciones y Estructura del Sistema
El sistema se organiza en torno al estado compartido del tablero y la gestión concurrente de jugadores a través de las siguientes estructuras y componentes clave:

* **Estructura del Tablero y Estado de Juego:** Matriz o grafo que representa las intersecciones del tablero (5x5 + trono) y almacena las primitivas de sincronización necesarias para garantizar la exclusión mutua durante el movimiento de las piezas.
* **Servidor Matchmaker (`servidor_matchmaker`):** Recibe N jugadores entrantes, los agrupa en parejas (Puma y Perros) y los asigna a las mesas disponibles. Controla de forma pasiva la cola de espera para no superar el límite de M mesas simultáneas.
* **Árbitro por Mesa (`juez_mesa`):** Hilo o proceso independiente encargado de inicializar el tablero de cada mesa, coordinar el inicio de la partida, validar periódicamente que los movimientos cumplan las reglas del juego (adyacencia y capturas) e imprimir o registrar la matriz de juego en tiempo real.
* **Control de Piezas (`hilo_puma` / `hilo_perro`):** Ejecutan la lógica de movimiento aleatorio con retardo (entre 50 y 300 ms). El Puma busca capturar 6 perros saltando sobre ellos hacia casillas libres, mientras que los Perros buscan acorralar al Puma sin dejarle movimientos válidos.
* **Manejador de Resiliencia y Desconexiones (`detectar_fallo` / `manejar_sigint`):** Detecta desconexiones prematuras de jugadores para otorgar la victoria por abandono al oponente. Al recibir la señal SIGINT (Ctrl+C), detiene el ingreso de nuevos clientes, finaliza ordenadamente las partidas activas y destruye todos los recursos compartidos.

---

## 3. Justificación de Decisiones de Diseño y Sincronización
* **Exclusión Mutua en el Tablero:** Para evitar condiciones de carrera cuando múltiples fichas intentan ocupar la misma casilla de forma simultánea, se utilizó un esquema de sincronización basado en mutexes por casilla o mutex global del tablero combinado con variables de condición (`pthread_mutex` / `pthread_cond`) y semáforos POSIX (`sem_t`). Esto garantiza que solo una ficha obtenga la posición en milisegundos.
* **Prevención de Deadlocks y Starvation:** Para evitar bloqueos mutuos cuando dos fichas intentan acceder a las mismas posiciones interconectadas, los bloqueos de casillas se adquieren siguiendo un orden estricto de identificadores de casillas (solicitud ordenada de recursos) o mediante intentos no bloqueantes con reintentos controlados (*backoff* aleatorio).
* **Liberación Limpia de Recursos (Cero Leaks):** El servidor principal registra un manejador de señales para SIGINT que intercepta la interrupción del usuario, notificando a las mesas activas, destruyendo los mutexes/semáforos y liberando la memoria compartida o descriptores de archivos antes de finalizar, asegurando la ausencia de recursos huérfanos en el sistema operativo.

---

## 4. Modo de Compilación y Uso

Para compilar el proyecto con las banderas estrictas requeridas (`gcc` o `g++`):
```bash
make  
