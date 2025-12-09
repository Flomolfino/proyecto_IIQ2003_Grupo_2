
# Proyecto semestral Fenómenos de Transporte  
Colector solar de losa de hormigón para ACS en San Pedro de Atacama

## 1. Descripción general del proyecto

Este repositorio contiene el código y los datos de un modelo numérico para estudiar un colector solar integrado en la losa de techo de una vivienda típica de San Pedro de Atacama.  
El sistema consiste en una losa de hormigón dentro de la cual se embebe un serpentín de agua. La radiación solar calienta la superficie exterior del techo y, por conducción y convección, transfiere calor al agua que circula por el serpentín. Esa agua se piensa usar como aporte a agua caliente sanitaria (ACS).

El proyecto se enmarca en la realidad climática del norte de Chile, donde la radiación solar es muy alta, pero las temperaturas nocturnas pueden ser bajas. Evaluar cómo responde térmicamente este tipo de colector permite estimar su potencial de ahorro energético y su aporte a los objetivos de sustentabilidad del país (mayor uso de energía solar, menor consumo de combustibles fósiles para ACS y reducción de emisiones).

En este repositorio se trabaja solo el modelo de transporte de energía en el techo y en el agua computacionalmente.

---

## 2. Sistema físico modelado

El sistema corresponde a una porción representativa del techo de una casa en San Pedro de Atacama. Se modela en 2D (coordenadas x–z) como un paralelepipedo dividido con tres capas:

- Capa 1: hormigón superior, expuesta al exterior y a la radiación solar.
- Capa 2: canal de agua (serpentín), adentro de las planchas de hormigón armado.
- Capa 3: hormigón inferior, hacia el interior de la vivienda.

La geometría usada en el modelo es:

- L: largo del tramo de colector en dirección x (proyección del serpentín).
- B1: espesor de la capa de hormigón superior.
- B2: espesor de la capa de agua.
- B3: espesor de la capa de hormigón inferior.

En términos de los espesores, las posiciones características en z son:

- z = 0: superficie exterior del techo.
- z = B1: interfaz hormigón superior – agua.
- z = B1 + B2: interfaz agua – hormigón inferior.
- z = B1 + B2 + B3: cara inferior de la losa.

De forma muy esquemática, el sistema se ve como una “sandwich” de hormigón–agua–hormigón, con flujo de agua a lo largo de x dentro de la capa de agua.

Los fenómenos de transporte considerados son:

- Conducción de calor en las tres capas sólidas (y en la propia capa de agua, en dirección z).
- Convección forzada en x dentro de la capa de agua, debido al movimiento del fluido.
- Convección y radiación efectivas en la superficie exterior: se modelan como un balance superficial que combina radiación solar incidente (G(t)), cambio de energía interna de la losa y convección con el aire ambiente a temperatura T∞(t).

Principales condiciones de borde (tal como se implementan en el código):

- En z = 0 (superficie exterior):
  - Balance de energía superficial con radiación solar y convección:
    - entrada de calor α·G(t)
    - salida o entrada de calor por convección h(T_superficie − T∞(t))
- En la interfaz hormigón–agua (z = B1):
  - Continuidad de temperatura: T1 = T2.
  - Continuidad de flujo de calor: k1 (∂T1/∂z) = k2 (∂T2/∂z).
- En la interfaz agua–hormigón inferior (z = B1 + B2):
  - Continuidad de temperatura: T2 = T3.
  - Continuidad de flujo: k2 (∂T2/∂z) = k3 (∂T3/∂z).
- En la cara inferior de la losa (z = B1 + B2 + B3):
  - Aislamiento térmico ideal: ∂T3/∂z = 0.
- En la entrada de agua (x = 0):
  - Temperatura fija del fluido: T2(x=0,z) = Tin.
- En la salida de agua (x = L):
  - Condición de gradiente cero en x (flujo de salida convectivo dominante) para la capa de agua.
- En los extremos laterales en x de las capas de hormigón:
  - Se supone comportamiento uniforme en x, por lo que se imponen condiciones consistentes con gradientes pequeños o simetría.

Los supuestos principales son:

- Régimen estacionario en cada “instante” de tiempo: para cada t se considera que el sistema ya alcanzó un estado estacionario asociado a los valores instantáneos de G(t) y T∞(t). Es decir, se resuelven varios problemas estacionarios “congelando” las condiciones climáticas cada 10 minutos.
- Propiedades termofísicas constantes por capa (k, ρ, cp) y representativas de hormigón típico y agua líquida.
- Flujo completamente desarrollado en la sección de agua: se prescribe un perfil parabólico en z para la velocidad en x.
- Flujo 2D (x–z), despreciando variaciones en la dirección transversal de la casa.

En la carpeta Figuras se incluyen gráficos que sintetizan estos campos de temperatura y permiten visualizar cómo se reparte el calor dentro de la losa y del agua para distintas configuraciones de espesor B1.

---

## 3. Método numérico utilizado

El modelo se implementa mediante un esquema de diferencias finitas en 2D:

- Se discretiza la dirección x en Nx nodos, desde x = 0 hasta x = L.
- La dirección z se discretiza en tres submallas, una para cada capa:
  - Nz1 nodos en [0, B1] (hormigón superior).
  - Nz2 nodos en [B1, B1 + B2] (agua).
  - Nz3 nodos en [B1 + B2, B1 + B2 + B3] (hormigón inferior).
- Cada nodo (i,j) se asocia a una ecuación de balance de energía estacionario.

En la capa de hormigón (capa 1 y 3) se resuelve la ecuación de conducción 1D en z, para cada x, con diferencias centrales:

- Interior de la capa:
  - k (T_{i,j+1} − 2T_{i,j} + T_{i,j−1}) / Δz² = 0
- En contorno z = 0 se aplica una condición tipo Robin (radiación solar + convección), discretizada con esquema de derivada en un solo lado de orden O(Δz²).
- En la cara inferior aislada se impone ∂T/∂z = 0 con un esquema de derivada en un solo lado.

En la capa de agua (capa 2) la ecuación considera conducción en x y z más el término de convección en x:

- ρ₂ cₚ₂ vₓ(z) · (∂T₂/∂x) = k₂ (∂²T₂/∂x² + ∂²T₂/∂z²)

Para discretizar:

- En x se usa un esquema "upwind" implícito en la dirección del flujo (T depende más del nodo “aguas arriba”), lo que mejora la estabilidad y evita oscilaciones numéricas.
- En z se mantienen diferencias centrales para la segunda derivada.

En las interfaces entre capas:

- Se impone continuidad de temperatura y de flujo mediante ecuaciones que mezclan nodos de ambas capas:
  - k₁(T_int − T_1prev)/Δz₁ + k₂(T_int − T_2next)/Δz₂ = 0 para la interfaz superior.
  - Definición análoga para la interfaz inferior agua–hormigón.

Todas estas ecuaciones se ensamblan en un gran sistema lineal de la forma:

- A · T_vec = b

donde:

- T_vec contiene todas las temperaturas (T₁, T₂, T₃) de todos los nodos.
- A es la matriz de coeficientes construida a partir de los balances y condiciones de borde.
- b incluye los términos fuente debidos a radiación solar, convección con el aire y temperatura de entrada del agua.

La resolución numérica se hace con `numpy.linalg.lstsq`, que entrega una solución de mínimos cuadrados para T_vec. Luego:

- Se re-arma la solución en matrices T1[x,z], T2[x,z] y T3[x,z].
- Se generan cortes y mapas de temperatura (perfiles en z en el centro del techo, mapas 2D en la capa de agua, evolución temporal del agua en el centro del colector, etc.).

La discretización por capas y el uso de interfaces explícitas permite capturar correctamente las discontinuidades de propiedades y la continuidad de temperatura y flujo, algo clave para representar la transferencia de calor en estructuras multicapa como una losa de hormigón con serpentín.

---

## 4. Estructura del repositorio e instrucciones de ejecución

La estructura actual del proyecto es:

- `README.md`  ← este archivo
- `Codigos/`
  - `CODIGOS_A_EJECUTAR.ipynb`
  - `README_Codigos.md`
- `Datos/`
  - `G(t)_T(t)_FdT.csv`
  - `README_Datos.md`
- `Figuras/`
  - `Imagen1.png` … `Imagen5.png`
  - `f1.png` … `f5.png`
  - `README_Figuras.md`

Las instrucciones para correr la simulación son:

1. Requisitos de software

   - Python 3.10 o superior.
   - Librerías:
     - `numpy`
     - `matplotlib`
     - `pandas`
     - `jupyter` (para abrir el notebook)

   Instalación típica (en un entorno virtual o en tu instalación local):

   ```bash
   pip install numpy matplotlib pandas jupyter


2. Obtener el repositorio

   * Clonar o descargar el repositorio y descomprimirlo en una carpeta local.

3. Verificar ubicación de los datos

   * El archivo con la serie temporal climática es:

     * `Datos/G(t)_T(t)_FdT.csv`
   * El notebook asume que este archivo está accesible desde la carpeta `Codigos/`. Si fuera necesario, ajustar la ruta en la línea donde se lee el CSV (por ejemplo, `../Datos/G(t)_T(t)_FdT.csv`).

4. Ejecutar el código principal

   * Abrir una terminal en la carpeta del proyecto.

   * Ingresar a la carpeta `Codigos/`:

     ```bash
     cd Codigos
     ```

   * Lanzar Jupyter Notebook (o JupyterLab):

     ```bash
     jupyter notebook
     ```

   * Abrir el archivo:

     * `CODIGOS_A_EJECUTAR.ipynb`

   * Ejecutar las celdas en orden:

     * Primero se cargan librerías y parámetros geométricos/físicos.
     * Luego se lee el archivo de datos climáticos (`G(t)_T(t)_FdT.csv`).
     * Después se arma el sistema lineal para un instante de tiempo (por ejemplo t ≈ 14:00) y se resuelve.
     * Finalmente se generan los gráficos y se guardan en la carpeta `Figuras/`.

5. Cambiar parámetros o escenarios

   * Para estudiar otro espesor de capa superior (por ejemplo B1 = 0,01 m):

     * Modificar el valor de `B1` en la sección de parámetros del notebook.
     * Ejecutar nuevamente las celdas que construyen la malla, el sistema lineal y las que generan gráficos.
   * Los resultados para B1 = 0,07 m se guardan como `ImagenX.png`; los resultados análogos para B1 = 0,01 m se guardan como `fX.png`.

---

## 5. Gráficos y resultados principales

Los resultados numéricos se resumen en la carpeta `Figuras/`. Hay dos grupos de archivos:

* `Imagen1.png` … `Imagen5.png`:

  * Conjunto de gráficos obtenidos para la configuración con capa superior de hormigón B1 = 0,07 m.
  * Incluyen:

    * Perfiles verticales de temperatura (T₁, T₂, T₃) en z en el punto medio del colector.
    * Mapas de temperatura en la capa de agua (T₂(x,z)) para un horario representativo (por ejemplo, alrededor de las 14:00).
    * Evolución temporal de la temperatura del agua en el centro del colector a lo largo de un día, usando la serie climática G(t) y T∞(t).

* `f1.png` … `f5.png`:

  * Mismos tipos de gráficos, pero para la configuración alternativa con B1 = 0,01 m.
  * Permiten comparar cómo cambia la inercia térmica del techo y el nivel de calentamiento del agua al reducir el espesor de la capa superior de hormigón.

Para más detalle sobre qué representa cada archivo de imagen se puede revisar:

* `Figuras/README_Figuras.md`

Interpretación física en el contexto de sustentabilidad:

* Los perfiles en z muestran cómo la radiación solar calienta primero la capa superior de hormigón y luego transfiere calor al agua y al resto de la losa.
* Los mapas de temperatura en la capa de agua permiten visualizar qué tan uniforme es el calentamiento a lo largo del serpentín y cuánto se calienta el fluido entre la entrada y la salida.
* Las curvas de temperatura en el tiempo evidencian la inercia térmica del sistema: el agua sigue relativamente caliente incluso después de que baja la radiación, lo que puede ser útil para disponer de ACS en horas de la tarde/noche.
* Comparar los casos B1 = 0,07 m y B1 = 0,01 m ayuda a discutir compromisos de diseño:

  * Con menor espesor de hormigón superior, la respuesta es más rápida y el agua se calienta más, pero se reduce la capacidad de almacenamiento térmico de la losa.
  * Con mayor espesor, la losa actúa más como “batería” térmica, suavizando las variaciones de temperatura pero con menores picos de calentamiento del agua.

Esta información sirve de base para evaluar, al menos cualitativamente, si un colector integrado en la losa de una vivienda en San Pedro de Atacama puede aportar de forma relevante a la demanda de ACS, y qué configuraciones son más prometedoras desde el punto de vista de eficiencia y confort térmico.

---
