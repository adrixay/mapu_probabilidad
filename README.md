"""
Agrupación de datos por el método de Sturges - Interfaz gráfica (Tkinter)
--------------------------------------------------------------------------
Esta es la versión ORIGINAL (sin histograma), comentada paso a paso.

Reglas implementadas:

1. El usuario ingresa los datos (separados por espacios, comas de lista o
   saltos de línea). Los decimales SOLO se aceptan con punto (.), nunca con
   coma (,). Si se detecta una coma decimal, se avisa al usuario.

2. El usuario ingresa "d": la diferencia mínima entre los datos (precisión).

3. Número de clases (Sturges):
        k = 1 + log2(n)
   Como "k" suele dar decimal (ej. 6.3, entre 6 y 7), se prueban los dos
   enteros más cercanos (piso y techo). Para decidir cuál usar se calcula,
   para cada candidato:
        C = (Rango + d) / k
   y se compara el número de decimales de C con el número de decimales que
   tienen los datos ingresados. Se elige el primer candidato (empezando por
   el k menor) cuyo C tenga IGUAL o MENOS decimales que los datos. C pasa a
   ser el ancho de clase.

4. Primer límite real:
        Li(1) = dato_menor - d/2
   Los siguientes límites reales se obtienen sumando C sucesivamente.

5. Tabla final: Clase, Li - Ls, marca de clase (xi), frecuencia absoluta (fi),
   frecuencia relativa (hi), frecuencia acumulada (Fi) y frecuencia relativa
   acumulada (Hi).
"""

import math
import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext

try:
    import pandas as pd
    PANDAS_DISPONIBLE = True
except ImportError:
    PANDAS_DISPONIBLE = False

try:
    import matplotlib
    matplotlib.use("TkAgg")
    from matplotlib.figure import Figure
    from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
    MATPLOTLIB_DISPONIBLE = True
except ImportError:
    MATPLOTLIB_DISPONIBLE = False


# ----------------------------------------------------------------------
# Funciones auxiliares
# ----------------------------------------------------------------------

def contar_decimales_texto(token: str) -> int:
    """Cuenta los decimales de un token de texto (tal como lo escribió el usuario).

    Es clave porque necesitamos saber cuántos decimales tienen los datos
    ORIGINALES (ej. "13.50" -> 2 decimales) para poder compararlos más
    adelante con los decimales del ancho de clase C.
    """
    if "." in token:
        return len(token.split(".")[1])
    return 0


def contar_decimales_valor(valor: float, tol: float = 1e-9, max_dec: int = 8) -> int:
    """Cuenta los decimales 'reales' que necesita un valor calculado (como C),
    evitando el ruido típico de la aritmética de punto flotante en Python
    (por ejemplo, que 0.3 se guarde internamente como 0.299999999999998...).

    Prueba redondear el valor a 0, 1, 2, ... decimales y se queda con el
    primer redondeo que reproduce el valor original dentro de una tolerancia.
    """
    for dec in range(0, max_dec + 1):
        if abs(round(valor, dec) - valor) < tol:
            return dec
    return max_dec


def separar_tokens(texto: str):
    """Convierte el texto ingresado en una lista de tokens numéricos (str).

    Aquí se hace la primera línea de defensa contra las comas decimales:
    si detecta el patrón dígito-coma-dígito (ej. "3,5"), lo interpreta como
    un error del usuario (coma decimal) y lanza una excepción explicando
    que debe usarse punto.
    """
    crudo = texto.strip()
    if not crudo:
        return []

    # Detectar posibles comas decimales: patrón dígito,dígito
    import re
    if re.search(r"\d,\d", crudo):
        raise ValueError(
            "Se detectó una coma como separador decimal (ej. 3,5). "
            "Los decimales deben escribirse con PUNTO (ej. 3.5)."
        )

    # Reemplazar comas (de lista) y saltos de línea por espacios
    crudo = crudo.replace(",", " ").replace("\n", " ").replace("\t", " ")
    tokens = [t for t in crudo.split(" ") if t.strip() != ""]
    return tokens


def validar_y_convertir(tokens):
    """Valida que cada token sea un número válido (solo punto decimal)."""
    valores = []
    for t in tokens:
        t = t.strip()
        # Solo se permiten dígitos, un punto y un signo negativo opcional
        if not es_numero_valido(t):
            raise ValueError(f"El dato '{t}' no es un número válido (use solo punto '.' para decimales).")
        valores.append(float(t))
    return valores


def es_numero_valido(token: str) -> bool:
    """Verifica que un token sea un número válido: solo dígitos, un único
    punto decimal opcional y un signo negativo opcional al inicio.
    Cualquier otro caracter (letras, comas, dos puntos, etc.) lo rechaza."""
    if token.count(".") > 1:
        return False
    cuerpo = token[1:] if token.startswith("-") else token
    if cuerpo == "":
        return False
    return all(c.isdigit() or c == "." for c in cuerpo) and cuerpo != "."


# ----------------------------------------------------------------------
# Lógica del método de Sturges (con las reglas particulares pedidas)
# ----------------------------------------------------------------------

class ResultadoSturges:
    """Simple 'contenedor' de datos: aquí se guarda TODO el proceso
    (candidatos evaluados, el k elegido, los límites, las clases, etc.)
    para que tanto el resumen como la tabla puedan mostrarlo sin tener
    que recalcular nada."""
    def __init__(self):
        self.n = 0
        self.minimo = 0.0
        self.maximo = 0.0
        self.rango = 0.0
        self.d = 0.0
        self.decimales_datos = 0
        self.k_float = 0.0
        self.candidatos = []       # lista de (k, C, decimales_C)
        self.k_elegido = 0
        self.C = 0.0
        self.uso_respaldo = False
        self.limites = []          # límites reales
        self.clases = []           # lista de dicts con la info de cada clase


def calcular_sturges(datos, d, decimales_datos) -> ResultadoSturges:
    """Corazón del programa: aplica el método de Sturges con las reglas
    particulares que pediste, en este orden exacto."""
    res = ResultadoSturges()
    res.n = len(datos)
    res.minimo = min(datos)
    res.maximo = max(datos)
    res.rango = res.maximo - res.minimo
    res.d = d
    res.decimales_datos = decimales_datos

    # PASO 1: k = 1 + log2(n) -> normalmente da decimal (ej. 4.9)
    res.k_float = 1 + math.log2(res.n)
    k_piso = math.floor(res.k_float)
    k_techo = math.ceil(res.k_float)

    # PASO 2: se generan los dos candidatos enteros (redondeo hacia abajo
    # y hacia arriba). Si k_float ya fuera entero, solo hay un candidato.
    candidatos_k = [k_piso] if k_piso == k_techo else [k_piso, k_techo]
    candidatos_k = [k for k in candidatos_k if k > 0]

    # PASO 3: para cada candidato se calcula C = (Rango + d) / k
    # y se cuenta cuántos decimales tiene ese C.
    for k in candidatos_k:
        C = (res.rango + res.d) / k
        dec_C = contar_decimales_valor(C)
        res.candidatos.append((k, C, dec_C))

    # PASO 4: se recorren los candidatos DEL K MENOR AL MAYOR y se elige
    # el primero cuyo C tenga decimales <= decimales de los datos originales.
    elegido = None
    for k, C, dec_C in res.candidatos:
        if dec_C <= res.decimales_datos:
            elegido = (k, C)
            break

    if elegido is None:
        # Si ningún candidato cumple exactamente la condición, se usa
        # el que tenga menos decimales y se redondea C como respaldo,
        # dejando la señal `uso_respaldo=True` para avisarlo en pantalla.
        k, C, dec_C = min(res.candidatos, key=lambda r: r[2])
        C = round(C, res.decimales_datos)
        elegido = (k, C)
        res.uso_respaldo = True

    res.k_elegido, res.C = elegido

    # PASO 5: primer límite real = dato_menor - d/2. Los siguientes límites
    # se obtienen sumando C sucesivamente, uno por cada clase.
    li0 = res.minimo - res.d / 2
    limites = [li0]
    for _ in range(res.k_elegido):
        limites.append(limites[-1] + res.C)
    res.limites = limites

    # PASO 6: se recorre cada dato y se ubica en su clase correspondiente.
    # Se usa [Li, Ls) para todas las clases, EXCEPTO la última, que se
    # cierra como [Li, Ls] para no perder el dato máximo (que coincide
    # justo con el límite superior de la última clase).
    eps = 1e-9
    conteos = [0] * res.k_elegido
    for x in datos:
        idx = None
        for i in range(res.k_elegido):
            li, ls = limites[i], limites[i + 1]
            es_ultima = (i == res.k_elegido - 1)
            if es_ultima:
                if (li - eps) <= x <= (ls + eps):
                    idx = i
                    break
            else:
                if (li - eps) <= x < (ls - eps):
                    idx = i
                    break
        if idx is None:
            # Red de seguridad (no debería ocurrir): si por algún redondeo
            # un dato no cae en ningún intervalo, se asigna a la clase
            # cuya marca de clase (xi) esté más cerca del dato.
            distancias = [abs(x - (limites[i] + limites[i + 1]) / 2) for i in range(res.k_elegido)]
            idx = distancias.index(min(distancias))
        conteos[idx] += 1

    # PASO 7: se arma cada fila de la tabla, acumulando Fi y Hi
    # a medida que se avanza clase por clase.
    Fi_acum = 0
    Hi_acum = 0.0
    for i in range(res.k_elegido):
        li, ls = limites[i], limites[i + 1]
        xi = (li + ls) / 2          # marca de clase = punto medio del intervalo
        fi = conteos[i]              # frecuencia absoluta = cuántos datos cayeron aquí
        hi = fi / res.n               # frecuencia relativa = fi / n
        Fi_acum += fi                 # frecuencia acumulada
        Hi_acum += hi                 # frecuencia relativa acumulada
        res.clases.append({
            "clase": i + 1,
            "li": li,
            "ls": ls,
            "xi": xi,
            "fi": fi,
            "hi": hi,
            "Fi": Fi_acum,
            "Hi": Hi_acum,
        })

    return res


# ----------------------------------------------------------------------
# Interfaz gráfica
# ----------------------------------------------------------------------

class App(tk.Tk):
    """Ventana principal de la interfaz gráfica. No hace cálculos por sí
    misma: solo recolecta lo que escribe el usuario, se lo pasa a
    calcular_sturges() y muestra el resultado (resumen y tabla)."""
    def __init__(self):
        super().__init__()
        self.title("Agrupación de datos - Método de Sturges")
        self.geometry("980x680")
        self.minsize(860, 600)
        self.configure(bg="#f2f4f7")

        self.ultimo_resultado = None  # guarda el ResultadoSturges más reciente
        self.df = None                # guarda la tabla de clases como DataFrame de pandas
        self._construir_widgets()

    # ------------------------------------------------------------------
    def _construir_widgets(self):
        estilo = ttk.Style(self)
        try:
            estilo.theme_use("clam")
        except tk.TclError:
            pass
        estilo.configure("Treeview.Heading", font=("Segoe UI", 9, "bold"))
        estilo.configure("Treeview", rowheight=24, font=("Segoe UI", 9))

        titulo = tk.Label(
            self, text="Agrupación de datos por el método de Sturges",
            font=("Segoe UI", 14, "bold"), bg="#f2f4f7", fg="#1f2937"
        )
        titulo.pack(pady=(12, 4))

        subtitulo = tk.Label(
            self,
            text="Ingrese los datos (separados por espacio, coma o salto de línea). "
                 "Use PUNTO para decimales, ej: 12.5",
            font=("Segoe UI", 9), bg="#f2f4f7", fg="#4b5563"
        )
        subtitulo.pack(pady=(0, 10))

        # --- Panel superior: entradas ---
        panel_entrada = tk.Frame(self, bg="#f2f4f7")
        panel_entrada.pack(fill="x", padx=16)

        # Caja de texto de datos
        frame_datos = tk.LabelFrame(panel_entrada, text="Datos", bg="#f2f4f7",
                                     font=("Segoe UI", 9, "bold"))
        frame_datos.pack(side="left", fill="both", expand=True, padx=(0, 10))

        self.txt_datos = scrolledtext.ScrolledText(frame_datos, height=6, width=50,
                                                     font=("Consolas", 10))
        self.txt_datos.pack(fill="both", expand=True, padx=6, pady=6)

        # Panel derecho: d, botones
        frame_derecha = tk.Frame(panel_entrada, bg="#f2f4f7")
        frame_derecha.pack(side="left", fill="y")

        frame_d = tk.LabelFrame(frame_derecha, text="Parámetro d\n(diferencia mínima entre datos)",
                                 bg="#f2f4f7", font=("Segoe UI", 9, "bold"))
        frame_d.pack(fill="x", pady=(0, 10))

        vcmd = (self.register(self._validar_entrada_numerica), "%P")
        self.entry_d = tk.Entry(frame_d, font=("Consolas", 11), width=15,
                                 validate="key", validatecommand=vcmd, justify="center")
        self.entry_d.pack(padx=8, pady=8)

        btn_frame = tk.Frame(frame_derecha, bg="#f2f4f7")
        btn_frame.pack(fill="x")

        self.btn_calcular = tk.Button(
            btn_frame, text="Calcular", command=self.procesar,
            bg="#2563eb", fg="white", font=("Segoe UI", 10, "bold"),
            activebackground="#1d4ed8", activeforeground="white",
            relief="flat", padx=12, pady=8, cursor="hand2"
        )
        self.btn_calcular.pack(fill="x", pady=(4, 4))

        self.btn_limpiar = tk.Button(
            btn_frame, text="Limpiar", command=self.limpiar,
            bg="#e5e7eb", fg="#111827", font=("Segoe UI", 10),
            relief="flat", padx=12, pady=8, cursor="hand2"
        )
        self.btn_limpiar.pack(fill="x", pady=(0, 4))

        self.btn_graficos = tk.Button(
            btn_frame, text="Ver Gráficos", command=self.mostrar_graficos,
            bg="#16a34a", fg="white", font=("Segoe UI", 10, "bold"),
            activebackground="#15803d", activeforeground="white",
            relief="flat", padx=12, pady=8, cursor="hand2"
        )
        self.btn_graficos.pack(fill="x")

        # --- Panel de resumen de cálculos ---
        frame_resumen = tk.LabelFrame(self, text="Resumen del cálculo", bg="#f2f4f7",
                                       font=("Segoe UI", 9, "bold"))
        frame_resumen.pack(fill="x", padx=16, pady=(12, 8))

        self.lbl_resumen = tk.Label(
            frame_resumen, text="Ingrese los datos y presione «Calcular».",
            justify="left", anchor="w", bg="#f2f4f7", fg="#111827",
            font=("Consolas", 9), wraplength=920
        )
        self.lbl_resumen.pack(fill="x", padx=8, pady=8)

        # --- Tabla de resultados ---
        frame_tabla = tk.LabelFrame(self, text="Tabla de frecuencias", bg="#f2f4f7",
                                     font=("Segoe UI", 9, "bold"))
        frame_tabla.pack(fill="both", expand=True, padx=16, pady=(0, 16))

        columnas = ("clase", "intervalo", "xi", "fi", "hi", "Fi", "Hi")
        self.tabla = ttk.Treeview(frame_tabla, columns=columnas, show="headings", height=10)

        encabezados = {
            "clase": "Clase",
            "intervalo": "Li - Ls",
            "xi": "xi (marca de clase)",
            "fi": "fi (frec. absoluta)",
            "hi": "hi (frec. relativa)",
            "Fi": "Fi (frec. acumulada)",
            "Hi": "Hi (frec. rel. acumulada)",
        }
        anchos = {
            "clase": 60, "intervalo": 190, "xi": 150,
            "fi": 130, "hi": 140, "Fi": 140, "Hi": 170,
        }
        for col in columnas:
            self.tabla.heading(col, text=encabezados[col])
            self.tabla.column(col, width=anchos[col], anchor="center")

        scrollbar = ttk.Scrollbar(frame_tabla, orient="vertical", command=self.tabla.yview)
        self.tabla.configure(yscrollcommand=scrollbar.set)

        self.tabla.pack(side="left", fill="both", expand=True, padx=(6, 0), pady=6)
        scrollbar.pack(side="left", fill="y", pady=6)

    # ------------------------------------------------------------------
    def _validar_entrada_numerica(self, texto_propuesto: str) -> bool:
        """Validador 'en vivo' del campo d: Tkinter llama a esta función
        cada vez que el usuario presiona una tecla en ese campo, ANTES de
        mostrar el nuevo carácter. Si devuelve False, el carácter se
        bloquea y nunca llega a aparecer en pantalla (así se evita
        físicamente escribir letras o comas)."""
        if texto_propuesto == "":
            return True
        if "," in texto_propuesto:
            return False
        if texto_propuesto.count(".") > 1:
            return False
        cuerpo = texto_propuesto
        return all(c.isdigit() or c == "." for c in cuerpo)

    # ------------------------------------------------------------------
    def limpiar(self):
        self.txt_datos.delete("1.0", tk.END)
        self.entry_d.delete(0, tk.END)
        self.lbl_resumen.config(text="Ingrese los datos y presione «Calcular».")
        for item in self.tabla.get_children():
            self.tabla.delete(item)
        self.ultimo_resultado = None
        self.df = None

    # ------------------------------------------------------------------
    def procesar(self):
        """Se ejecuta al presionar 'Calcular'. Orquesta todo el flujo:
        1) lee y valida los datos y el valor de d,
        2) llama a calcular_sturges() (aquí NO se hace ningún cálculo,
           solo se delega a la función matemática),
        3) actualiza el resumen y la tabla en pantalla."""
        texto_datos = self.txt_datos.get("1.0", tk.END)
        texto_d = self.entry_d.get().strip()

        # --- Validar datos ---
        try:
            tokens = separar_tokens(texto_datos)
        except ValueError as e:
            messagebox.showerror("Error en los datos", str(e))
            return

        if len(tokens) < 2:
            messagebox.showerror("Error en los datos", "Ingrese al menos 2 datos.")
            return

        try:
            datos = validar_y_convertir(tokens)
        except ValueError as e:
            messagebox.showerror("Error en los datos", str(e))
            return

        # --- Validar d ---
        if texto_d == "":
            messagebox.showerror("Error en 'd'", "Debe ingresar el valor de d.")
            return
        if not es_numero_valido(texto_d):
            messagebox.showerror("Error en 'd'", "El valor de d no es válido (use solo punto para decimales).")
            return
        d = float(texto_d)
        if d <= 0:
            messagebox.showerror("Error en 'd'", "El valor de d debe ser mayor que 0.")
            return

        decimales_datos = max(contar_decimales_texto(t) for t in tokens)

        # --- Calcular ---
        try:
            res = calcular_sturges(datos, d, decimales_datos)
        except Exception as e:
            messagebox.showerror("Error en el cálculo", f"Ocurrió un error: {e}")
            return

        self.ultimo_resultado = res
        # El DataFrame se arma SOLO con los datos que ya calculó calcular_sturges
        # (res.clases). Pandas aquí no agrupa ni recalcula nada, solo organiza
        # la misma información en una tabla para poder graficarla fácilmente.
        if PANDAS_DISPONIBLE:
            self.df = pd.DataFrame(res.clases)
        else:
            self.df = None

        self._mostrar_resumen(res, decimales_datos)
        self._mostrar_tabla(res)

    # ------------------------------------------------------------------
    def _mostrar_resumen(self, res: ResultadoSturges, decimales_datos: int):
        """Solo formatea texto: toma lo que ya calculó calcular_sturges()
        y arma la cadena que se ve arriba de la tabla (n, rango, k, C, etc.).
        No hace ningún cálculo nuevo."""
        cand_txt = "; ".join(
            f"k={k} -> C={C:.6g} ({dec} decimales)" for k, C, dec in res.candidatos
        )
        aviso_respaldo = ""
        if res.uso_respaldo:
            aviso_respaldo = ("\n⚠ Ningún candidato cumplió exactamente la condición de decimales; "
                               "se usó el mejor candidato y se redondeó C.")

        texto = (
            f"n = {res.n}   |   mínimo = {res.minimo:g}   |   máximo = {res.maximo:g}   |   "
            f"Rango (R) = {res.rango:g}   |   d = {res.d:g}   |   decimales de los datos = {decimales_datos}\n"
            f"k = 1 + log2({res.n}) = {res.k_float:.4f}\n"
            f"Candidatos evaluados: {cand_txt}\n"
            f"→ k elegido = {res.k_elegido}   |   C (ancho de intervalo) = {res.C:g}\n"
            f"Primer límite real: Li(1) = {res.minimo:g} - {res.d:g}/2 = {res.limites[0]:g}"
            f"{aviso_respaldo}"
        )
        self.lbl_resumen.config(text=texto)

    # ------------------------------------------------------------------
    def _mostrar_tabla(self, res: ResultadoSturges):
        """Recorre res.clases (ya calculado) y llena cada fila del Treeview.
        Tampoco calcula nada: solo da formato con la cantidad de decimales
        correcta (`dec`) para que se vea prolijo."""
        for item in self.tabla.get_children():
            self.tabla.delete(item)

        dec = res.decimales_datos if res.decimales_datos > 0 else 2

        for c in res.clases:
            intervalo = f"[{c['li']:.{dec}f}  -  {c['ls']:.{dec}f}]"
            self.tabla.insert("", "end", values=(
                c["clase"],
                intervalo,
                f"{c['xi']:.{dec}f}",
                c["fi"],
                f"{c['hi']:.4f}",
                c["Fi"],
                f"{c['Hi']:.4f}",
            ))


    # ------------------------------------------------------------------
    def mostrar_graficos(self):
        """Abre una ventana nueva con tres pestañas (Histograma, Polígono
        de frecuencias y Ojiva). Los tres se construyen leyendo columnas
        del DataFrame de pandas (self.df), que a su vez viene directo de
        res.clases: NO se vuelve a agrupar ni a recalcular nada aquí,
        solo se grafica lo que calculó calcular_sturges()."""
        if not MATPLOTLIB_DISPONIBLE:
            messagebox.showerror(
                "Falta matplotlib",
                "Para ver los gráficos se necesita la librería matplotlib.\n"
                "Instálala con:  pip install matplotlib"
            )
            return
        if not PANDAS_DISPONIBLE:
            messagebox.showerror(
                "Falta pandas",
                "Para ver los gráficos se necesita la librería pandas.\n"
                "Instálala con:  pip install pandas"
            )
            return

        res = self.ultimo_resultado
        if res is None or self.df is None:
            messagebox.showwarning("Sin datos", "Primero presiona «Calcular» para generar la tabla.")
            return

        dec = res.decimales_datos if res.decimales_datos > 0 else 2
        df = self.df  # columnas: clase, li, ls, xi, fi, hi, Fi, Hi

        ventana = tk.Toplevel(self)
        ventana.title("Gráficos - Método de Sturges")
        ventana.geometry("820x600")
        ventana.configure(bg="white")

        notebook = ttk.Notebook(ventana)
        notebook.pack(fill="both", expand=True, padx=8, pady=8)

        tab_hist = tk.Frame(notebook, bg="white")
        tab_poligono = tk.Frame(notebook, bg="white")
        tab_ojiva = tk.Frame(notebook, bg="white")
        notebook.add(tab_hist, text="Histograma")
        notebook.add(tab_poligono, text="Polígono de frecuencias")
        notebook.add(tab_ojiva, text="Ojiva")

        # ---------------- Histograma ----------------
        fig_hist = Figure(figsize=(7.4, 5.0), dpi=100)
        ax_hist = fig_hist.add_subplot(111)
        barras = ax_hist.bar(df["xi"], df["fi"], width=res.C, align="center",
                              color="#2563eb", edgecolor="black", linewidth=0.8)
        for barra, valor in zip(barras, df["fi"]):
            ax_hist.annotate(str(valor),
                              xy=(barra.get_x() + barra.get_width() / 2, barra.get_height()),
                              xytext=(0, 3), textcoords="offset points",
                              ha="center", va="bottom", fontsize=9)
        ax_hist.set_xticks(df["xi"])
        ax_hist.set_xticklabels([f"{v:.{dec}f}" for v in df["xi"]], fontsize=8)
        ax_hist.set_xlabel("Marca de clase (xi)")
        ax_hist.set_ylabel("Frecuencia absoluta (fi)")
        ax_hist.set_title("Histograma de frecuencias")
        ax_hist.spines["top"].set_visible(False)
        ax_hist.spines["right"].set_visible(False)
        fig_hist.tight_layout()
        canvas_hist = FigureCanvasTkAgg(fig_hist, master=tab_hist)
        canvas_hist.draw()
        canvas_hist.get_tk_widget().pack(fill="both", expand=True, padx=6, pady=6)

        # ---------------- Polígono de frecuencias ----------------
        # Se "cierra" el polígono agregando un punto en 0 antes de la primera
        # clase y otro en 0 después de la última (a una distancia C), como
        # se hace tradicionalmente en estadística.
        xi_poligono = [df["xi"].iloc[0] - res.C] + list(df["xi"]) + [df["xi"].iloc[-1] + res.C]
        fi_poligono = [0] + list(df["fi"]) + [0]

        fig_pol = Figure(figsize=(7.4, 5.0), dpi=100)
        ax_pol = fig_pol.add_subplot(111)
        ax_pol.plot(xi_poligono, fi_poligono, marker="o", color="#dc2626",
                    linewidth=2, markerfacecolor="white", markeredgewidth=2)
        ax_pol.set_xlabel("Marca de clase (xi)")
        ax_pol.set_ylabel("Frecuencia absoluta (fi)")
        ax_pol.set_title("Polígono de frecuencias")
        ax_pol.spines["top"].set_visible(False)
        ax_pol.spines["right"].set_visible(False)
        fig_pol.tight_layout()
        canvas_pol = FigureCanvasTkAgg(fig_pol, master=tab_poligono)
        canvas_pol.draw()
        canvas_pol.get_tk_widget().pack(fill="both", expand=True, padx=6, pady=6)

        # ---------------- Ojiva (frecuencia acumulada) ----------------
        # Se grafica Fi contra el límite superior (ls) de cada clase, y se
        # agrega el punto inicial (primer límite real, Fi=0) para que la
        # curva empiece desde cero.
        x_ojiva = [res.limites[0]] + list(df["ls"])
        y_ojiva = [0] + list(df["Fi"])

        fig_ojiva = Figure(figsize=(7.4, 5.0), dpi=100)
        ax_ojiva = fig_ojiva.add_subplot(111)
        ax_ojiva.plot(x_ojiva, y_ojiva, marker="o", color="#16a34a",
                      linewidth=2, markerfacecolor="white", markeredgewidth=2)
        ax_ojiva.set_xlabel("Límite superior de clase (Ls)")
        ax_ojiva.set_ylabel("Frecuencia acumulada (Fi)")
        ax_ojiva.set_title("Ojiva (frecuencia acumulada 'menos que')")
        ax_ojiva.spines["top"].set_visible(False)
        ax_ojiva.spines["right"].set_visible(False)
        fig_ojiva.tight_layout()
        canvas_ojiva = FigureCanvasTkAgg(fig_ojiva, master=tab_ojiva)
        canvas_ojiva.draw()
        canvas_ojiva.get_tk_widget().pack(fill="both", expand=True, padx=6, pady=6)


if __name__ == "__main__":
    app = App()
    app.mainloop()
