# Proyecto final EDA I: Daemon de enrutamiento dinámico para redes mesh de alta disponibilidad en C

## Índice

1. [Introducción](#Introducción)

2. [Justificación Técnica](#justificación-técnica) 

   1. Justificación del proyecto

   2. Justificación del lenguaje

   3. Justificación de algoritmos

3. [Arquitectura de memoria y uso de estructuras de datos](#arquitectura-de-memoria-y-uso-de-estructuras-de-datos) 

   1. Infraestructura persistente

      - Tabla hash

      - Vértices

      - Aristas

   2. Algoritmos y estructuras efímeras

      - Cola de prioridad

      - Pila

4. [Redundancia y mitigación de riesgos](#redundancia-y-mitigación-de-riesgos) 

5. [Fases de desarrollo](#fases-de-desarrollo-planeadas)

## Introducción

## Justificación técnica

### Justificación del proyecto

El presente proyecto se diseña y plantea para resolver la volatilidad de los enlaces de radiofrecuencia en redes comunitarias inalámbricas (*mesh*), operando sobre hardware embebido con recursos de cómputo severamente restringidos (procesadores MIPS/ARM a $<500\text{ MHz}$ y $32\text{ MB}$ a $64\text{ MB}$ de RAM).

A diferencia de las redes cableadas tradicionales donde la topología es estática y predecible, los enlaces aéreos en una red comunitaria sufren degradación constante por interferencia electromagnética en bandas libres (2.4/5 GHz), obstáculos físicos móviles y la desconexión fortuita de nodos. El *daemon* proporciona un mecanismo de control de Capa 3 autónomo, reactivo y determinista que mapea la topología en tiempo real, evalúa el costo de los enlaces mediante una métrica combinada de latencia y pérdida de paquetes, y calcula las rutas óptimas con una huella de memoria RAM mínima.

Al centralizar el cómputo en el procesador local mediante estructuras de datos eficientes y una arquitectura basada en Estado de Enlace, el *daemon* evita la inundación constante de paquetes de *broadcast* (ARP/DHCP) que colapsa el espectro aéreo en soluciones de Capa 2, garantizando la escalabilidad y estabilidad de la red troncal sin depender de infraestructura ni proveedores centralizados.

> Cambiar la redacción

### Justificación del lenguaje

El desarrollo del *daemon* se realiza estrictamente en **C puro (estándar C99/C11)** debido a las restricciones físicas del hardware embebido (routers de azotea con procesadores MIPS/ARM y memorias de $32\text{ MB}$ a $64\text{ MB}$) y a los requerimientos de tiempo real del protocolo de red:

- **Control determinista de memoria:** La ausencia de un recolector de basura (*Garbage Collector*) elimina las pausas impredecibles del sistema (*Stop-the-World*). La gestión explícita mediante `malloc` y `free` garantiza que el cálculo de rutas y la actualización de tablas respondan con latencias acotadas en microsegundos ante caídas de enlace.

- **Eficiencia de registro y huella de RAM mínima:** C permite la manipulación directa de bits, el uso de tipos de ancho fijo (`uint32_t`, `uint16_t`) y la alineación estricta de estructuras (`struct`). Esto reduce la representación de una dirección IPv4 a solo $4\text{ bytes}$ en RAM, permitiendo almacenar miles de nodos y aristas en un par de megabytes.

- **Acceso nativo a las APIs del Kernel de Linux:** Permite interactuar directamente con la pila de red del sistema operativo mediante sockets `NETLINK` (para inyectar rutas en la tabla FIB) y el subsistema de registros (`syslog`), sin la sobrecarga computacional ni la latencia que imponen las capas de abstracción o bibliotecas de terceros.

- **Ejecución reactiva y bajo consumo térmico:** El código compilado a lenguaje máquina directo permite al proceso permanecer en reposo ($0\%$ de uso de CPU) y despertarse instantáneamente mediante eventos de red (*sockets I/O*), maximizando la vida útil del hardware embebido expuesto a condiciones intemperies.

> Cambiar  la redacción

### Justificación de  algoritmos

#### Algoritmo  de exploración

¿Porqué Dijsktra y no  B.A.T.M.A.N.?

Dijsktra conoce toda la  red y la guarda en memoria, recalcula toda si se agrega un  nuevo nodo o cada  x tiempo, complejidad o(ElogV)

B.A.T.M.A.N. trabaja siempre a ciegas  y se basa en encontrar  el nodo  al que es  más barato  de ir  solo en  sus  vecinos, complejidad  O(1), no  guarda  memoria  y  calcula cada  paquete  al instante

#### Algoritmo de ordenamiento

*Insertion sort*

```c
void sortedInsert(struct Node** head_ref, struct Node* new_node) {
    struct Node* current;
    
    // Caso base: Lista vacía o el nodo debe ir al inicio
    if (*head_ref == NULL || (*head_ref)->data >= new_node->data) {
        new_node->next = *head_ref;
        *head_ref = new_node;
    } else {
        // Recorrer hasta encontrar la posición correcta
        current = *head_ref;
        while (current->next != NULL && current->next->data < new_node->data) {
            current = current->next;
        }
        new_node->next = current->next;
        current->next = new_node;
    }
}

// Función principal: Ordena la lista enlazada modificando los punteros
void insertionSort(struct Node** head_ref) {
    struct Node* sorted = NULL; // Lista auxiliar inicialmente vacía
    struct Node* current = *head_ref;
    
    while (current != NULL) {
        struct Node* next = current->next; // Guarda el resto de la lista
        
        sortedInsert(&sorted, current); // Inserta el nodo actual en la lista ordenada
        
        current = next; // Avanza al siguiente nodo original
    }
    
    *head_ref = sorted; // Apunta la cabeza original a la lista ya ordenada
}
```

El problema principal al resolver es el ordenamiento  dinámico  y secuencial de conexiones basado  en  la métrica  combinada de latencia y perdida de  paquetes, debido a que  la  implementación se hace sobre una lista simple y no sobre un arreglo  no  se  necesita generar  ningun costo en desplazamientos  de  bytes, además  debido  al tamaño de la muestra de datos, ninguno de  los algoritmos  de ordenamiento  más  avanzados pierden todas  sus ventajas, además  de  que al  no  necesitar la creación de  copias es el algoritmo ideal  para entornos con recursos de  hardware muy  limitados.

#### Algorítmo de  hasheo

*MurmurHash3*
Cómo funciona: Aplica 3 operaciones de desplazo de bits (*XOR Shift*) mezcladas con multiplicaciones por constantes primo-mágicas sobre el entero de 32 bits antes de aplicar el módulo ($\text{hash} \pmod N$). 
Además produce efecto  avalancha

> Completar justificación

## Arquitectura de memoria y uso de estructuras de datos

### Infraestructura persistente

- **Tabla hash** mediante arreglos de manera que al hashearse el mismo se genera un arreglo el cual crece dinámicamente en función de el número de nodos que se encuentran en ella, se hace por medio de números primos 

- **Vértice (AP)** cada router físico es un vértice dentro del grafo es decir cada vértice debe guardar: 

  - id único (IP) 

  - Indicador de si es repetidor o es un gateway

  - un apuntador a todos sus vecinos

  - un apuntador al siguiente nodo en caso de que se manden 2 hashes

Estructura propuesta

```c
typedef struct Vertice
{
 char ip[16];
 bool gateway;
 
 struct Nodo *nextCol
 struct Arista *neighbors
}Vertice;
```

- **Arista (Enlaces)** cada conexión es una arista y todas nacen de un vértice dentro del grafo, se guarda:

  - id del destino (IP)

  - latencia regulada para que no se colapse

  - porcentaje de pérdida de paquétes

  - peso de la métrica calculado con la latencia y pérdida

  - apuntador a la siguiente arista

Estructura propuesta

```c
typedef struct Arista 
{
 char ipDest;
 float latencyMS;
 float lostRate;
 float Metric
 
 sctruct Arista *siguiente
}Arista;
```

### Algoritmos y  estructuras efímeras

- __Dikjstra__

- __Cola de prioriad__

## Redundancia y mitigación de riesgos

Para  la mitigación de riesgos

## Fases de desarrollo planeadas

- Generar la tabla mesh dinámica, que  crezca con base  al número de nodos que  se  inserten además  de poder  insertar y buscar ip's.

- Agregar la  lista de  vecinos aun nodo y que sea ordenada  al insertarse  por  métrica.

- Pruebas de  reversión de la ruta  para posterior  aplicación con Dijkstra

- Implementar  la inserción y extración  del elemento  con menor peso  en la cola  de prioridad

- Aplicar algoritmo para poder calcular la  ruta  óptima entre  un  origen y  un  destino

- Cargar  un  arcivo  para poder  leer la topología de  la  red  sin necesitar hardcodearla

- Capas de  resiliencia

  - Control de  TTL para evitar bucles  infinitos de paquetes

  - Poda LRU de  nodos  inactivos cuando la memoria alcance  un índice  de alta  ocupación

  - Desviación dinámica por congestión

- Comunicación  con Linux


