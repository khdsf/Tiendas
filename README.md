import ttkbootstrap as tb

import tkinter as tk
from tkinter import messagebox, simpledialog, filedialog
import sqlite3
import os
import calendar
import datetime
import matplotlib.pyplot as plt
import cv2
from pyzbar.pyzbar import decode	
from PIL import Image, ImageTk
import barcode
from barcode.writer import ImageWriter

# (Opcional) para la báscula
try:
    import serial
except ImportError:
    serial = None

# [MEJORA] LOGGING
import logging
import hashlib  # Para hash de contraseñas si se requiere

# [MEJORA] USO DE HASH DE CONTRASEÑAS (activar/desactivar)
USE_HASHED_PASSWORDS = False

def custom_round(value):
    """Redondea el valor si la parte decimal >= 0.3."""
    integer_part = int(value)
    decimal_part = value - integer_part
    return integer_part + 1 if decimal_part >= 0.3 else integer_part

# [MEJORA] Funciones auxiliares para hash de contraseñas
def hash_password(password: str) -> str:
    """
    Retorna el hash SHA-256 en formato hexadecimal de la contraseña dada.
    """
    return hashlib.sha256(password.encode('utf-8')).hexdigest()

def check_password(password: str, stored_hash: str) -> bool:
    """
    Verifica si la contraseña coincide con el hash guardado.
    """
    return hash_password(password) == stored_hash

# ------------------ LOGIN ------------------
def show_login(root, db_manager, app):
    login_win = tb.Toplevel(root)
    login_win.title("Login")
    login_win.geometry("500x300")
    login_win.grab_set()

    tb.Label(login_win, text="Usuario:", font=("Segoe UI", 12)).pack(pady=10)
    entry_user = tb.Entry(login_win)
    entry_user.pack(pady=5)

    tb.Label(login_win, text="Contraseña:", font=("Segoe UI", 12)).pack(pady=10)
    entry_pass = tb.Entry(login_win, show="*")
    entry_pass.pack(pady=5)

    def attempt_login():
        user = entry_user.get().strip()
        pwd = entry_pass.get().strip()
        if not user or not pwd:
            messagebox.showerror("Login", "Ingrese usuario y contraseña.", parent=login_win)
            logging.warning("Intento de login con campos vacíos")  # [MEJORA] LOG
            return

        # [MEJORA] Dependiendo si usamos contraseñas hasheadas
        if USE_HASHED_PASSWORDS:
            cursor = db_manager.execute("SELECT * FROM users WHERE username=?", (user,))
            row = cursor.fetchone() if cursor else None
            if row:
                # row[2] = 'password' o 'hash'
                stored_pw = row[2] or ""
                # Verificamos con check_password
                if check_password(pwd, stored_pw):
                    # Login correcto
                    role = row[3]
                    if role in ("limpieza", "carnicero"):
                        messagebox.showerror("Login", "Este usuario no tiene acceso a la app.", parent=login_win)
                        logging.info(f"Usuario {user} (rol {role}) sin acceso a la app.")
                        return

                    app.current_user_role = role
                    app.current_username = row[1]
                    app.login_time = datetime.datetime.now()  # Hora de login

                    logging.info(f"Usuario '{user}' logeado correctamente con rol '{role}'")  # [MEJORA] LOG
                    messagebox.showinfo("Login", f"Rol asignado a: {role}", parent=login_win)
                    login_win.destroy()
                    app.setup_all_tabs()  # Construir toda la interfaz
                else:
                    logging.warning(f"Login fallido para usuario '{user}'")  # [MEJORA] LOG
                    messagebox.showerror("Login", "Credenciales incorrectas.", parent=login_win)
            else:
                logging.warning(f"Usuario '{user}' no encontrado en la BD")  # [MEJORA] LOG
                messagebox.showerror("Login", "Credenciales incorrectas.", parent=login_win)
        else:
            # Modo anterior, sin hash
            cursor = db_manager.execute("SELECT * FROM users WHERE username=? AND password=?", (user, pwd))
            row = cursor.fetchone() if cursor else None
            if row:
                role = row[3]
                if role in ("limpieza","carnicero"):
                    messagebox.showerror("Login", "Este usuario no tiene acceso a la app.", parent=login_win)
                    logging.info(f"Usuario {user} (rol {role}) sin acceso a la app.")
                    return

                app.current_user_role = role
                app.current_username = row[1]
                app.login_time = datetime.datetime.now()  # Hora de login

                logging.info(f"Usuario '{user}' logeado correctamente con rol '{role}'")  # [MEJORA] LOG
                messagebox.showinfo("Login", f"Rol asignado a: {role}", parent=login_win)
                login_win.destroy()
                app.setup_all_tabs()  # Construir toda la interfaz
            else:
                logging.warning(f"Login fallido para usuario '{user}'")  # [MEJORA] LOG
                messagebox.showerror("Login", "Credenciales incorrectas.", parent=login_win)

    tb.Button(login_win, text="Login", command=attempt_login, bootstyle="primary").pack(pady=10)
    login_win.wait_window()

# ------------------ BD ------------------
class DatabaseManager:
    def __init__(self, db_path="tienda.db"):
        self.db_path = db_path
        self.conn = None
        self.cursor = None
        self.connect()
        self.create_tables()
        self.ensure_default_users()

    def connect(self):
        try:
            self.conn = sqlite3.connect(self.db_path)
            self.cursor = self.conn.cursor()
        except Exception as e:
            messagebox.showerror("Error de Conexión", f"No se pudo conectar a la base de datos: {e}")
            logging.error(f"Error de conexión a la BD: {e}")  # [MEJORA] LOG

    def create_tables(self):
        try:
            self.cursor.execute('''
                CREATE TABLE IF NOT EXISTS products (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    code TEXT UNIQUE,
                    name TEXT,
                    category TEXT,
                    stock INTEGER,
                    price REAL,
                    image_path TEXT
                )
            ''')
            self.cursor.execute('''
                CREATE TABLE IF NOT EXISTS users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    username TEXT UNIQUE,
                    password TEXT,
                    role TEXT
                )
            ''')
            self.conn.commit()
        except Exception as e:
            messagebox.showerror("Error", f"Error al crear las tablas: {e}")
            logging.error(f"Error al crear tablas: {e}")  # [MEJORA] LOG

    def ensure_default_users(self):
        """
        [MEJORA] Si las contraseñas usan hash, se guardará encriptado.
        De lo contrario, se guarda el texto plano como antes.
        """
        self.cursor.execute("SELECT COUNT(*) FROM users")
        if self.cursor.fetchone()[0] == 0:
            # Usuarios por defecto
            def_pw_admin  = hash_password("admin") if USE_HASHED_PASSWORDS else "admin"
            def_pw_encarg = hash_password("encargado") if USE_HASHED_PASSWORDS else "encargado"
            def_pw_cajero = hash_password("cajero") if USE_HASHED_PASSWORDS else "cajero"
            def_pw_limp   = hash_password("") if USE_HASHED_PASSWORDS else ""
            def_pw_carn   = hash_password("") if USE_HASHED_PASSWORDS else ""

            self.cursor.execute("INSERT INTO users (username, password, role) VALUES (?,?,?)", ("admin",     def_pw_admin,  "admin"))
            self.cursor.execute("INSERT INTO users (username, password, role) VALUES (?,?,?)", ("encargado", def_pw_encarg, "encargado"))
            self.cursor.execute("INSERT INTO users (username, password, role) VALUES (?,?,?)", ("cajero",    def_pw_cajero, "cajero"))
            self.cursor.execute("INSERT INTO users (username, password, role) VALUES (?,?,?)", ("limpieza1", def_pw_limp,   "limpieza"))
            self.cursor.execute("INSERT INTO users (username, password, role) VALUES (?,?,?)", ("carnicero1",def_pw_carn,   "carnicero"))
            self.conn.commit()
            logging.info("Usuarios por defecto creados")  # [MEJORA] LOG

    def execute(self, query, params=()):
        try:
            c = self.conn.cursor()
            c.execute(query, params)
            self.conn.commit()
            return c
        except Exception as e:
            messagebox.showerror("Error en BD", str(e))
            logging.error(f"Error en BD: {e}")  # [MEJORA] LOG
            return None

# ------------------ APP ------------------
class BarcodeApp:
    def __init__(self, root, db_manager):
        self.root = root
        self.db_manager = db_manager
        self.current_user_role = None
        self.current_username = None
        self.login_time = None  # Hora de login

        # Márgenes de ganancia
        self.general_margin = 1.3
        self.fv_margin = 1.3

        # Configuración de ventana
        root.title("Tienda los primos")
        root.geometry("1000x700")

        # [MEJORA] Ajustamos el estilo general para que se vea más grande y fácil de leer.
        self.style = tb.Style("cosmo")
        self.style.configure('.', font=("Segoe UI", 12))

        # [MEJORA] Llamamos a un método para configurar logging
        self.configure_logging()

        # Variables
        self.setup_variables()

        # Creamos el estilo y widgets principales
        self.configure_excel_style()
        self.create_widgets()

        logging.info("Aplicación iniciada correctamente")  # [MEJORA] LOG

    # [MEJORA] Método para configurar el logging
    def configure_logging(self):
        logging.basicConfig(
            level=logging.INFO,  # Se puede cambiar a DEBUG para más detalle
            format="%(asctime)s [%(levelname)s] %(message)s",
            datefmt="%Y-%m-%d %H:%M:%S"
        )

    def setup_variables(self):
        # Ventas
        self.account_items = []
        self.total_amount = 0
        self.surtidores_expenses = []

        # Calendario
        self.calendar_notes = {}
        self.payment_info = {}
        self.weekly_payments = {}
        self.monthly_rents = {}

    def configure_excel_style(self):
        """
        Configuramos un estilo "Excel.Treeview" para que tenga
        líneas tipo Excel en las tablas, usando el tema actual ("cosmo").
        """
        style = self.style
        # Creamos un estilo personalizado
        style.configure(
            "Excel.Treeview",
            background="white",
            foreground="black",
            rowheight=25,
            fieldbackground="white",
            bordercolor="#cccccc",
            borderwidth=1,
            relief="solid"
        )
        style.map(
            "Excel.Treeview",
            background=[("selected", "#bfe")],
            foreground=[("selected", "black")]
        )
        style.layout(
            "Excel.Treeview",
            [("Excel.Treeview.treearea", {"sticky": "nswe"})]
        )

    def create_widgets(self):
        # Frame superior para albergar el reloj y el botón de Cerrar Sesión
        topbar_frame = tb.Frame(self.root)
        topbar_frame.pack(side="top", fill="x")

        # Reloj (arriba a la derecha)
        self.clock_label = tb.Label(topbar_frame, text="", font=("Segoe UI",12,"bold"))
        self.clock_label.pack(side="right", anchor="e", padx=10, pady=5)
        self.update_clock()

        # Botón de Cerrar Sesión
        logout_button = tb.Button(
            topbar_frame,
            text="Cerrar Sesión",
            command=self.logout,
            bootstyle="danger"
        )
        logout_button.pack(side="left", padx=10, pady=5)

        # Notebook principal
        self.notebook = tb.Notebook(self.root)
        self.notebook.pack(fill="both", expand=True, padx=10, pady=10)

        # Pestañas
        self.frame_sales = tb.Frame(self.notebook)
        self.notebook.add(self.frame_sales, text="Ventas")

        self.frame_inventory = tb.Frame(self.notebook)
        self.notebook.add(self.frame_inventory, text="Inventario")

        self.frame_calendar = tb.Frame(self.notebook)
        self.notebook.add(self.frame_calendar, text="Calendario")

        self.frame_reports = tb.Frame(self.notebook)
        self.notebook.add(self.frame_reports, text="Reportes")

        self.frame_users = tb.Frame(self.notebook)
        self.notebook.add(self.frame_users, text="Usuarios")

        self.frame_config = tb.Frame(self.notebook)
        self.notebook.add(self.frame_config, text="Configuración")

    def logout(self):
        if messagebox.askyesno("Cerrar Sesión", "¿Seguro que desea cerrar sesión?"):
            logging.info(f"Usuario {self.current_username} cerró sesión")  # [MEJORA] LOG
            self.root.destroy()

    def update_clock(self):
        now = datetime.datetime.now().strftime("%H:%M:%S")
        self.clock_label.config(text=now)
        self.root.after(1000, self.update_clock)

    def setup_all_tabs(self):
        """Se llama tras el login, para construir la lógica de cada pestaña."""
        self.setup_sales_tab()
        self.setup_inventory_tab()
        self.setup_calendar_tab()
        self.setup_reports_tab()
        self.setup_users_tab()
        self.setup_config_tab()

        self.apply_role_permissions()
        self.notebook.select(self.frame_sales)

    def apply_role_permissions(self):
        if self.current_user_role in ("admin","encargado"):
            pass
        elif self.current_user_role=="cajero":
            self.notebook.tab(self.frame_users, state="hidden")
        else:
            pass

    # ------------------ VENTAS ------------------
    def setup_sales_tab(self):
        # Búsqueda
        top_frame = tb.Frame(self.frame_sales)
        top_frame.pack(fill="x", pady=5)

        tb.Label(top_frame, text="Buscar producto:", font=("Segoe UI",12)).pack(side="left", padx=5)
        self.entry_search_sales = tb.Entry(top_frame)
        self.entry_search_sales.pack(side="left", padx=5)

        self.combo_cat_sales = tb.Combobox(top_frame, state="readonly")
        self.combo_cat_sales.pack(side="left", padx=5)
        cats = self.get_all_categories()
        cats.insert(0,"Todas")
        self.combo_cat_sales["values"] = cats
        self.combo_cat_sales.current(0)

        btn_search = tb.Button(top_frame, text="Buscar Producto", command=self.search_and_add_sales, bootstyle="info")
        btn_search.pack(side="left", padx=5)

        # Botones +Verdura / +Fruta (solo admin/encargado)
        if self.current_user_role in ("admin","encargado"):
            admin_frame = tb.Frame(self.frame_sales)
            admin_frame.pack(side="top", anchor="e", pady=5)
            tb.Button(admin_frame, text="+ Verdura", command=self.add_new_verdura, bootstyle="success").pack(side="left", padx=5)
            tb.Button(admin_frame, text="+ Fruta", command=self.add_new_fruta, bootstyle="success").pack(side="left", padx=5)

        # Contenedor principal
        container = tb.Frame(self.frame_sales)
        container.pack(fill="both", expand=True, padx=10, pady=10)
        container.grid_columnconfigure(0, weight=1)
        container.grid_columnconfigure(1, weight=2)
        container.grid_columnconfigure(2, weight=1)
        container.grid_rowconfigure(0, weight=1)

        left_frame = tb.Frame(container)
        left_frame.grid(row=0, column=0, sticky="nsew", padx=5, pady=5)
        center_frame = tb.Frame(container)
        center_frame.grid(row=0, column=1, sticky="nsew", padx=5, pady=5)
        right_frame = tb.Frame(container)
        right_frame.grid(row=0, column=2, sticky="nsew", padx=5, pady=5)

        # Tabla de Ventas (usando estilo "Excel.Treeview")
        self.tree_sales = tb.Treeview(
            center_frame,
            columns=("Producto","Descripción","Cantidad","Precio Unitario","Subtotal"),
            show="headings",
            style="Excel.Treeview"
        )
        for col in ("Producto","Descripción","Cantidad","Precio Unitario","Subtotal"):
            self.tree_sales.heading(col, text=col)
            self.tree_sales.column(col, width=100)
        self.tree_sales.pack(fill="both", expand=True)

        self.lbl_total = tb.Label(center_frame, text="Cuenta Total: $0.00", font=("Segoe UI",12,"bold"))
        self.lbl_total.pack(pady=5)

        # Botones abajo
        bottom_frame = tb.Frame(self.frame_sales)
        bottom_frame.pack(side="bottom", pady=5)

        tb.Button(bottom_frame, text="Eliminar Venta", command=self.remove_sale, bootstyle="danger").pack(side="left", padx=5)
        tb.Button(bottom_frame, text="Agregar a cuenta", command=self.add_manual_sale, bootstyle="success").pack(side="left", padx=5)
        tb.Button(bottom_frame, text="Agregar gasto en surtidores", command=self.add_surtidor_expense, bootstyle="info").pack(side="left", padx=5)
        tb.Button(bottom_frame, text="Cobrar", command=self.cobrar, bootstyle="primary").pack(side="left", padx=5)

        # Paneles de verduras y frutas
        self.draw_verduras(left_frame)
        self.draw_frutas(right_frame)

    def draw_verduras(self, parent):
        sample = [
            {"nombre":"Tomate","precio":20.0},
            {"nombre":"Cebolla","precio":15.0}
        ]
        for item in sample:
            f = tb.Frame(parent, borderwidth=2, relief="groove")
            f.pack(pady=5, fill="x")
            tb.Label(f, text=item["nombre"]).pack()
            btn = tb.Button(f, text="Agregar", command=lambda d=item: self.add_simple_sale(d))
            btn.pack()

    def draw_frutas(self, parent):
        sample = [
            {"nombre":"Manzana","precio":25.0},
            {"nombre":"Plátano","precio":18.0}
        ]
        for item in sample:
            f = tb.Frame(parent, borderwidth=2, relief="groove")
            f.pack(pady=5, fill="x")
            tb.Label(f, text=item["nombre"]).pack()
            btn = tb.Button(f, text="Agregar", command=lambda d=item: self.add_simple_sale(d))
            btn.pack()

    def add_simple_sale(self, data):
        cant = 1
        subtotal = cant * data["precio"]
        self.tree_sales.insert("", tk.END, values=(data["nombre"],"",cant,f"${data['precio']:.2f}",f"${subtotal:.2f}"))
        self.account_items.append({"name":data["nombre"],"desc":"","cantidad":cant,"precio_unit":data["precio"],"subtotal":subtotal})
        self.total_amount += subtotal
        self.lbl_total.config(text=f"Cuenta Total: ${self.total_amount:.2f}")

    def add_new_verdura(self):
        # Solo admin/encargado
        if self.current_user_role not in ("admin","encargado"):
            messagebox.showerror("Permiso denegado","Solo admin/encargado puede agregar verduras.")
            return
        win = tb.Toplevel(self.root)
        win.title("Agregar Verdura")
        win.geometry("400x300")

        tb.Label(win, text="Nombre de la verdura:", font=("Segoe UI",12)).pack(pady=5)
        entry_name = tb.Entry(win)
        entry_name.pack(pady=5)

        tb.Label(win, text="Precio (por kg):", font=("Segoe UI",12)).pack(pady=5)
        entry_price = tb.Entry(win)
        entry_price.pack(pady=5)

        tb.Label(win, text="Ruta de Imagen:", font=("Segoe UI",12)).pack(pady=5)
        entry_image = tb.Entry(win)
        entry_image.pack(pady=5)

        def pick_image():
            path = filedialog.askopenfilename(title="Seleccionar Imagen", filetypes=[("PNG","*.png"),("Todos","*.*")])
            if path:
                entry_image.delete(0, tk.END)
                entry_image.insert(0, path)
        tb.Button(win, text="Buscar Imagen", command=pick_image, bootstyle="info").pack(pady=5)

        def save_verdura():
            name = entry_name.get().strip()
            if not name:
                messagebox.showerror("Error","El nombre es obligatorio.",parent=win)
                return
            try:
                price = float(entry_price.get().strip())
            except:
                messagebox.showerror("Error","Precio inválido.",parent=win)
                return
            messagebox.showinfo("Agregar","Verdura agregada.",parent=win)
            logging.info(f"Verdura agregada: {name} (Precio: {price})")  # [MEJORA] LOG
            win.destroy()
            self.setup_sales_tab()

        tb.Button(win, text="Agregar", command=save_verdura, bootstyle="success").pack(pady=10)

    def add_new_fruta(self):
        # Solo admin/encargado
        if self.current_user_role not in ("admin","encargado"):
            messagebox.showerror("Permiso denegado","Solo admin/encargado puede agregar frutas.")
            return
        win = tb.Toplevel(self.root)
        win.title("Agregar Fruta")
        win.geometry("400x300")

        tb.Label(win, text="Nombre de la fruta:", font=("Segoe UI",12)).pack(pady=5)
        entry_name = tb.Entry(win)
        entry_name.pack(pady=5)

        tb.Label(win, text="Precio (por kg):", font=("Segoe UI",12)).pack(pady=5)
        entry_price = tb.Entry(win)
        entry_price.pack(pady=5)

        tb.Label(win, text="Ruta de Imagen:", font=("Segoe UI",12)).pack(pady=5)
        entry_image = tb.Entry(win)
        entry_image.pack(pady=5)

        def pick_image():
            path = filedialog.askopenfilename(title="Seleccionar Imagen", filetypes=[("PNG","*.png"),("Todos","*.*")])
            if path:
                entry_image.delete(0, tk.END)
                entry_image.insert(0, path)
        tb.Button(win, text="Buscar Imagen", command=pick_image, bootstyle="info").pack(pady=5)

        def save_fruta():
            name = entry_name.get().strip()
            if not name:
                messagebox.showerror("Error","El nombre es obligatorio.",parent=win)
                return
            try:
                price = float(entry_price.get().strip())
            except:
                messagebox.showerror("Error","Precio inválido.",parent=win)
                return
            messagebox.showinfo("Agregar","Fruta agregada.",parent=win)
            logging.info(f"Fruta agregada: {name} (Precio: {price})")  # [MEJORA] LOG
            win.destroy()
            self.setup_sales_tab()

        tb.Button(win, text="Agregar", command=save_fruta, bootstyle="success").pack(pady=10)

    def search_and_add_sales(self):
        text = self.entry_search_sales.get().strip()
        cat = self.combo_cat_sales.get().strip()
        query = "SELECT * FROM products WHERE (name LIKE ? OR code LIKE ?)"
        params = [f"%{text}%", f"%{text}%"]
        if cat!="Todas":
            query += " AND category=?"
            params.append(cat)
        c = self.db_manager.execute(query, params)
        results = c.fetchall() if c else []
        if not results:
            messagebox.showinfo("Ventas","No se encontró ningún producto.")
            return
        if len(results)==1:
            self.add_product_to_sales(results[0])
        else:
            self.show_multiple_results_sales(results)

    def show_multiple_results_sales(self, results):
        top = tb.Toplevel(self.root)
        top.title("Coincidencias")
        top.geometry("400x300")

        lb = tk.Listbox(top)
        lb.pack(fill="both", expand=True)

        for row in results:
            display = f"{row[2]} (Cod:{row[1]}) - ${row[5]:.2f}"
            lb.insert("end", display)

        def on_sel():
            idx = lb.curselection()
            if not idx:
                return
            chosen = results[idx[0]]
            self.add_product_to_sales(chosen)
            top.destroy()

        tb.Button(top, text="Agregar", command=on_sel, bootstyle="success").pack(pady=5)

    def add_product_to_sales(self, product_row):
        name = product_row[2]
        price = product_row[5]
        cant = 1
        subtotal = cant*price
        self.tree_sales.insert("", tk.END, values=(name,"",cant,f"${price:.2f}",f"${subtotal:.2f}"))
        self.account_items.append({"name":name,"desc":"","cantidad":cant,"precio_unit":price,"subtotal":subtotal})
        self.total_amount += subtotal
        self.lbl_total.config(text=f"Cuenta Total: ${self.total_amount:.2f}")

    def remove_sale(self):
        selection = self.tree_sales.selection()
        if not selection:
            messagebox.showwarning("Eliminar Venta","Seleccione una venta para eliminar.")
            return
        for sel in selection:
            vals = self.tree_sales.item(sel,"values")
            try:
                st = float(vals[4].replace("$",""))
            except:
                st = 0
            self.total_amount -= st
            self.tree_sales.delete(sel)
        self.lbl_total.config(text=f"Cuenta Total: ${self.total_amount:.2f}")
        messagebox.showinfo("Eliminar Venta","Venta(s) eliminada(s).")

    def add_manual_sale(self):
        win = tb.Toplevel(self.root)
        win.title("Agregar a cuenta")
        win.geometry("400x300")

        tb.Label(win, text="Nombre del producto:", font=("Segoe UI",12)).pack(pady=5)
        entry_name = tb.Entry(win)
        entry_name.pack(pady=5)

        tb.Label(win, text="Descripción (opcional):", font=("Segoe UI",12)).pack(pady=5)
        entry_desc = tb.Entry(win)
        entry_desc.pack(pady=5)

        tb.Label(win, text="Cantidad o Peso:", font=("Segoe UI",12)).pack(pady=5)
        entry_cant = tb.Entry(win)
        entry_cant.pack(pady=5)

        tb.Label(win, text="Precio Unitario:", font=("Segoe UI",12)).pack(pady=5)
        entry_precio = tb.Entry(win)
        entry_precio.pack(pady=5)

        def save_manual():
            name = entry_name.get().strip()
            desc = entry_desc.get().strip()
            try:
                cant = float(entry_cant.get().strip())
            except:
                messagebox.showerror("Error","Cantidad/Peso inválido.",parent=win)
                return
            try:
                pu = float(entry_precio.get().strip())
            except:
                messagebox.showerror("Error","Precio Unitario inválido.",parent=win)
                return
            if not name:
                messagebox.showerror("Error","El nombre es obligatorio.",parent=win)
                return
            subtotal = cant*pu
            self.tree_sales.insert("", tk.END, values=(name,desc,cant,f"${pu:.2f}",f"${subtotal:.2f}"))
            self.account_items.append({"name":name,"desc":desc,"cantidad":cant,"precio_unit":pu,"subtotal":subtotal})
            self.total_amount += subtotal
            self.lbl_total.config(text=f"Cuenta Total: ${self.total_amount:.2f}")
            messagebox.showinfo("Agregar a cuenta","Producto agregado manualmente a la cuenta.",parent=win)
            logging.info(f"Venta manual agregada: {name} x {cant} @ {pu}")  # [MEJORA] LOG
            win.destroy()

        tb.Button(win, text="Agregar", command=save_manual, bootstyle="success").pack(pady=10)

    def add_surtidor_expense(self):
        win = tb.Toplevel(self.root)
        win.title("Agregar Gasto en Surtidores")
        win.geometry("400x200")

        tb.Label(win, text="Nombre del surtidor:", font=("Segoe UI",12)).pack(pady=5)
        entry_nombre = tb.Entry(win)
        entry_nombre.pack(pady=5)

        tb.Label(win, text="Gasto:", font=("Segoe UI",12)).pack(pady=5)
        entry_gasto = tb.Entry(win)
        entry_gasto.pack(pady=5)

        def save_expense():
            nombre = entry_nombre.get().strip()
            try:
                gasto = float(entry_gasto.get().strip())
            except:
                messagebox.showerror("Error","Gasto inválido.",parent=win)
                return
            if not nombre:
                messagebox.showerror("Error","El nombre del surtidor es obligatorio.",parent=win)
                return
            self.surtidores_expenses.append((nombre,gasto))
            messagebox.showinfo("Gasto agregado", f"Gasto para {nombre}: ${gasto:.2f}", parent=win)
            logging.info(f"Gasto surtidor agregado: {nombre} - ${gasto:.2f}")  # [MEJORA] LOG
            win.destroy()

        tb.Button(win, text="Agregar Gasto", command=save_expense, bootstyle="success").pack(pady=10)

    def cobrar(self):
        if self.total_amount<=0:
            messagebox.showinfo("Cobrar","No hay nada en la cuenta.")
            return
        win = tb.Toplevel(self.root)
        win.title("Cobrar")
        win.geometry("300x200")

        tb.Label(win, text=f"Total: ${self.total_amount:.2f}", font=("Segoe UI",12)).pack(pady=5)
        tb.Label(win, text="Monto pagado:", font=("Segoe UI",12)).pack(pady=5)
        entry_pago = tb.Entry(win)
        entry_pago.pack(pady=5)

        def do_cobrar():
            try:
                pago = float(entry_pago.get().strip())
            except:
                messagebox.showerror("Error","Monto inválido.")
                return
            if pago<self.total_amount:
                messagebox.showerror("Error","Pago insuficiente.")
                return
            cambio = pago - self.total_amount
            messagebox.showinfo("Cambio", f"Su cambio es ${cambio:.2f}")

            imprimir_ticket = messagebox.askyesno("Imprimir Ticket","¿Desea imprimir el ticket de la venta?", parent=win)
            if imprimir_ticket:
                self.generar_ticket_detallado()

            # Reiniciar la cuenta
            self.tree_sales.delete(*self.tree_sales.get_children())
            self.account_items.clear()
            self.total_amount=0
            self.lbl_total.config(text="Cuenta Total: $0.00")
            logging.info(f"Cobro realizado. Pago: {pago:.2f}, Cambio: {cambio:.2f}")  # [MEJORA] LOG
            win.destroy()

        tb.Button(win, text="Cobrar", command=do_cobrar, bootstyle="success").pack(pady=10)

    def generar_ticket_detallado(self):
        ticket_num = "000012"
        fecha_hora = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"ticket_{fecha_hora}.txt"

        cajero = self.current_username if self.current_username else "Desconocido"

        try:
            with open(filename, "w", encoding="utf-8") as f:
                f.write("================= TICKET DE VENTA =================\n")
                f.write("Tienda: Tienda los primos\n")
                f.write(f"Ticket #{ticket_num}\n")
                f.write(f"Fecha/Hora: {datetime.datetime.now()}\n")
                f.write(f"Cajero: {cajero}\n\n")

                f.write("Productos:\n")
                total_items = 0
                for i, item in enumerate(self.account_items, start=1):
                    total_items += 1
                    f.write(f"{i}) {item['name']}\n")
                    f.write(f"   Cantidad: {item['cantidad']}\n")
                    f.write(f"   Precio Unit.: ${item['precio_unit']:.2f}\n")
                    f.write(f"   Subtotal: ${item['subtotal']:.2f}\n\n")

                f.write("---------------------------------------------------\n")
                f.write(f"Total de Ítems: {total_items}\n")
                f.write(f"TOTAL A PAGAR: ${self.total_amount:.2f}\n")
                f.write("---------------------------------------------------\n")
                f.write("        ¡GRACIAS POR SU COMPRA!\n")
                f.write("===================================================\n")

            messagebox.showinfo("Ticket", f"Ticket guardado en {filename}")
            logging.info(f"Ticket generado: {filename}")  # [MEJORA] LOG
        except Exception as e:
            messagebox.showerror("Error Ticket", f"No se pudo generar el ticket: {e}")
            logging.error(f"Error al generar ticket: {e}")  # [MEJORA] LOG

    def get_all_categories(self):
        cats=[]
        c=self.db_manager.execute("SELECT DISTINCT category FROM products")
        if c:
            for row in c.fetchall():
                if row[0]:
                    cats.append(row[0])
        return cats

    # ------------------ INVENTARIO ------------------
    def setup_inventory_tab(self):
        for w in self.frame_inventory.winfo_children():
            w.destroy()

        lbl_title=tb.Label(self.frame_inventory, text="Gestión de Inventario", font=("Segoe UI",16,"bold"))
        lbl_title.pack(pady=15)

        self.inv_notebook=tb.Notebook(self.frame_inventory)
        self.inv_notebook.pack(fill="both", expand=True, padx=10, pady=10)

        self.frame_inv_general=tb.Frame(self.inv_notebook)
        self.inv_notebook.add(self.frame_inv_general, text="General")

        self.frame_inv_fv=tb.Frame(self.inv_notebook)
        self.inv_notebook.add(self.frame_inv_fv, text="Frutas y Verduras")

        self.frame_inv_stock=tb.Frame(self.inv_notebook)
        self.inv_notebook.add(self.frame_inv_stock, text="Estado de Stock")

        self.setup_inventory_general_tab()
        self.setup_inventory_fv_tab()
        self.setup_inventory_stock_tab()

        self.inv_notebook.select(self.frame_inv_general)

    def setup_inventory_general_tab(self):
        lbl_title=tb.Label(self.frame_inv_general, text="Inventario General", font=("Segoe UI",14,"bold"))
        lbl_title.pack(pady=5)

        search_frame = tb.Frame(self.frame_inv_general)
        search_frame.pack(pady=5)

        tb.Label(search_frame, text="Buscar producto:").pack(side="left", padx=5)
        self.entry_search_inventory=tb.Entry(search_frame)
        self.entry_search_inventory.pack(side="left", padx=5)

        self.combo_cat_inventory=tb.Combobox(search_frame, state="readonly")
        self.combo_cat_inventory.pack(side="left", padx=5)
        cat_list=self.get_all_categories()
        cat_list.insert(0,"Todas")
        self.combo_cat_inventory["values"]=cat_list
        self.combo_cat_inventory.current(0)

        btn_search=tb.Button(search_frame, text="Buscar", command=self.search_inventory, bootstyle="primary")
        btn_search.pack(side="left", padx=5)
        btn_show_all=tb.Button(search_frame, text="Mostrar Todo", command=self.load_inventory, bootstyle="primary")
        btn_show_all.pack(side="left", padx=5)

        columns=("ID","Código","Nombre","Categoría","Stock","Precio","Imagen")
        self.tree_inventory=tb.Treeview(self.frame_inv_general, columns=columns, show="headings", style="Excel.Treeview")
        for col in columns:
            self.tree_inventory.heading(col, text=col)
            self.tree_inventory.column(col, width=100)
        self.tree_inventory.pack(fill="both", expand=True, padx=10, pady=10)

        btn_frame=tb.Frame(self.frame_inv_general)
        btn_frame.pack(pady=10)
        tb.Button(btn_frame, text="Agregar Producto", command=self.add_product, bootstyle="success").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Editar Producto", command=self.edit_product, bootstyle="warning").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Eliminar Producto", command=self.delete_product, bootstyle="danger").pack(side="left", padx=5)

        self.load_inventory()

    def search_inventory(self):
        text=self.entry_search_inventory.get().strip()
        cat=self.combo_cat_inventory.get().strip()
        query="SELECT * FROM products WHERE (name LIKE ? OR code LIKE ?)"
        params=[f"%{text}%", f"%{text}%"]
        if cat!="Todas":
            query+=" AND category=?"
            params.append(cat)
        for row in self.tree_inventory.get_children():
            self.tree_inventory.delete(row)
        c=self.db_manager.execute(query, params)
        if c:
            for r in c.fetchall():
                self.tree_inventory.insert("", "end", values=r)

    def load_inventory(self):
        for row in self.tree_inventory.get_children():
            self.tree_inventory.delete(row)
        c=self.db_manager.execute("SELECT * FROM products")
        if c:
            for r in c.fetchall():
                self.tree_inventory.insert("", "end", values=r)

    def add_product(self):
        AddProductWindow(self)

    def edit_product(self):
        sel=self.tree_inventory.selection()
        if not sel:
            messagebox.showwarning("Editar Producto","Seleccione un producto para editar.")
            return
        data=self.tree_inventory.item(sel[0])["values"]
        EditProductWindow(self, data)

    def delete_product(self):
        sel=self.tree_inventory.selection()
        if not sel:
            messagebox.showwarning("Eliminar Producto","Seleccione un producto para eliminar.")
            return
        data=self.tree_inventory.item(sel[0])["values"]
        if messagebox.askyesno("Eliminar Producto", f"¿Está seguro de eliminar el producto {data[2]}?"):
            try:
                self.db_manager.execute("DELETE FROM products WHERE id=?", (data[0],))
                messagebox.showinfo("Eliminar Producto","Producto eliminado correctamente.")
                logging.info(f"Producto eliminado: {data[2]} (ID: {data[0]})")  # [MEJORA] LOG
                self.load_inventory()
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo eliminar el producto: {e}")
                logging.error(f"No se pudo eliminar el producto {data[2]}: {e}")  # [MEJORA] LOG

    def setup_inventory_fv_tab(self):
        lbl_info=tb.Label(self.frame_inv_fv, text="Editar precios de Frutas y Verduras", font=("Segoe UI",12,"bold"))
        lbl_info.pack(pady=5)

        self.tree_fv=tb.Treeview(self.frame_inv_fv, columns=("ID","Código","Nombre","Categoría","Stock","Precio"),
                                 show="headings", style="Excel.Treeview")
        for col in ("ID","Código","Nombre","Categoría","Stock","Precio"):
            self.tree_fv.heading(col, text=col)
            self.tree_fv.column(col, width=100)
        self.tree_fv.pack(fill="both", expand=True, padx=10, pady=10)

        btn_mod_price=tb.Button(self.frame_inv_fv, text="Modificar Precio", command=self.modify_fv_price, bootstyle="success")
        btn_mod_price.pack(pady=10)

        self.load_fv_inventory()

    def load_fv_inventory(self):
        for row in self.tree_fv.get_children():
            self.tree_fv.delete(row)
        c=self.db_manager.execute("SELECT id,code,name,category,stock,price FROM products WHERE category IN ('Fruta','Verdura')")
        if c:
            for r in c.fetchall():
                self.tree_fv.insert("", "end", values=r)

    def modify_fv_price(self):
        sel=self.tree_fv.selection()
        if not sel:
            messagebox.showwarning("Editar Precio","Seleccione un producto.")
            return
        data=self.tree_fv.item(sel[0])["values"]
        prod_id=data[0]
        current_price=data[5]

        win=tb.Toplevel(self.root)
        win.title("Modificar Precio")
        win.geometry("300x200")

        tb.Label(win, text=f"Producto: {data[2]}", font=("Segoe UI",12)).pack(pady=5)
        tb.Label(win, text=f"Precio Actual: {current_price:.2f}", font=("Segoe UI",12)).pack(pady=5)

        tb.Label(win, text="Nuevo Precio:", font=("Segoe UI",12)).pack(pady=5)
        entry_price=tb.Entry(win)
        entry_price.pack(pady=5)

        def save_new_price():
            try:
                new_p=float(entry_price.get().strip())
            except:
                messagebox.showerror("Error","Precio inválido.",parent=win)
                return
            self.db_manager.execute("UPDATE products SET price=? WHERE id=?", (new_p, prod_id))
            messagebox.showinfo("Modificar Precio","Precio actualizado.",parent=win)
            logging.info(f"Precio modificado para {data[2]} (ID: {prod_id}): {new_p}")  # [MEJORA] LOG
            win.destroy()
            self.load_fv_inventory()

        tb.Button(win, text="Guardar", command=save_new_price, bootstyle="success").pack(pady=10)

    def setup_inventory_stock_tab(self):
        lbl_info=tb.Label(self.frame_inv_stock, text="Estado de Stock (Rojo <=3, Amarillo <=6, Verde >6)", font=("Segoe UI",12,"bold"))
        lbl_info.pack(pady=5)

        self.tree_stock_status=tb.Treeview(self.frame_inv_stock, columns=("ID","Código","Nombre","Categoría","Stock","Precio"),
                                           show="headings", style="Excel.Treeview")
        for col in ("ID","Código","Nombre","Categoría","Stock","Precio"):
            self.tree_stock_status.heading(col, text=col)
            self.tree_stock_status.column(col, width=100)
        self.tree_stock_status.pack(fill="both", expand=True, padx=10, pady=10)

        self.tree_stock_status.tag_configure("low", foreground="red")     # <=3
        self.tree_stock_status.tag_configure("mid", foreground="orange")  # <=6
        self.tree_stock_status.tag_configure("high", foreground="green")  # >6

        btn_refresh=tb.Button(self.frame_inv_stock, text="Actualizar Estado", command=self.load_stock_status, bootstyle="info")
        btn_refresh.pack(pady=10)

        self.load_stock_status()

    def load_stock_status(self):
        for row in self.tree_stock_status.get_children():
            self.tree_stock_status.delete(row)

        c=self.db_manager.execute("SELECT id, code, name, category, stock, price FROM products")
        if c:
            for r in c.fetchall():
                stock_val = r[4]
                if stock_val <= 3:
                    tag_used = "low"
                elif stock_val <= 6:
                    tag_used = "mid"
                else:
                    tag_used = "high"
                self.tree_stock_status.insert("", "end", values=r, tags=(tag_used,))

    # ------------------ CALENDARIO ------------------
    def setup_calendar_tab(self):
        for w in self.frame_calendar.winfo_children():
            w.destroy()

        self.cal_toolbar = tb.Frame(self.frame_calendar)
        self.cal_toolbar.pack(side="top", fill="x")

        tb.Button(self.cal_toolbar, text="Semana Actual", command=self.show_week_view, bootstyle="info").pack(side="left", padx=5, pady=5)
        tb.Button(self.cal_toolbar, text="Meses", command=self.show_months_view, bootstyle="info").pack(side="left", padx=5, pady=5)

        tb.Button(self.cal_toolbar, text="Pago Semanal", command=self.record_weekly_payment, bootstyle="primary").pack(side="right", padx=5, pady=5)
        tb.Button(self.cal_toolbar, text="Renta Mensual", command=self.record_monthly_rent, bootstyle="primary").pack(side="right", padx=5, pady=5)

        self.cal_content = tb.Frame(self.frame_calendar)
        self.cal_content.pack(fill="both", expand=True)

        self.show_week_view()

    def show_week_view(self):
        for w in self.cal_content.winfo_children():
            w.destroy()

        today = datetime.date.today()
        current_weekday = today.weekday()
        monday = today - datetime.timedelta(days=current_weekday)

        for i in range(7):
            day = monday + datetime.timedelta(days=i)
            day_frame = tb.Frame(self.cal_content, borderwidth=2, relief="solid")
            day_frame.grid(row=0, column=i, padx=5, pady=5, ipadx=50, ipady=50, sticky="nsew")

            day_str = day.strftime("%A %d/%m/%Y")
            notes = self.calendar_notes.get((day.year, day.month, day.day), [])
            text_notes = ""
            if notes:
                for (n, d) in notes:
                    text_notes += f"\n{n}: {d}"

            payment_text = ""
            if (day.year, day.month, day.day) in self.payment_info:
                payment_text = f"\nPago: ${self.payment_info[(day.year, day.month, day.day)]:.2f}"

            lbl_text = f"{day_str}{text_notes}{payment_text}"
            lbl_day = tb.Label(day_frame, text=lbl_text, font=("Segoe UI", 10), justify="left")
            lbl_day.pack(padx=5, pady=5)

            if self.current_user_role in ("admin","encargado"):
                btn_det = tb.Button(day_frame, text="Detalles", command=lambda d=day: self.open_day_dialog(d))
                btn_det.pack(pady=5)

    def open_day_dialog(self, day):
        if self.current_user_role not in ("admin","encargado"):
            messagebox.showerror("Permiso denegado","No puedes editar este día.")
            return

        date_str = day.strftime("%d/%m/%Y")
        notes = self.calendar_notes.get((day.year, day.month, day.day), [])
        current_notes_str = "\n".join([f"{n}: {d}" for (n,d) in notes]) if notes else "Ninguna"
        add_note = messagebox.askyesno("Notas para "+date_str, f"Notas:\n{current_notes_str}\n¿Agregar nota?")
        if add_note:
            while True:
                note_name = simpledialog.askstring("Agregar Nota","Nombre de la nota (blanco para terminar):", parent=self.root)
                if not note_name:
                    break
                note_desc = simpledialog.askstring("Agregar Nota","Descripción de la nota:", parent=self.root) or ""
                notes.append((note_name, note_desc))
                if not messagebox.askyesno("Agregar Nota","¿Desea agregar otra nota?", parent=self.root):
                    break
            self.calendar_notes[(day.year, day.month, day.day)] = notes
            logging.info(f"Notas agregadas para {date_str}: {notes}")  # [MEJORA] LOG

        # Si es viernes(4) o sábado(5)
        if day.weekday() in (4,5):
            payment_str = simpledialog.askstring("Día de Pago "+date_str,"Monto pagado (dejar vacío si no hay pago):", parent=self.root)
            if payment_str:
                try:
                    payment = float(payment_str)
                    self.payment_info[(day.year, day.month, day.day)] = payment
                    logging.info(f"Pago registrado para {date_str}: {payment}")  # [MEJORA] LOG
                except:
                    messagebox.showerror("Error","Monto de pago inválido.")
        self.show_week_view()

    def show_months_view(self):
        for w in self.cal_content.winfo_children():
            w.destroy()

        current_year = datetime.date.today().year
        for i in range(12):
            month = i+1
            row = i//4
            col = i%4
            frame_month = tb.Frame(self.cal_content, borderwidth=2, relief="solid")
            frame_month.grid(row=row, column=col, padx=5, pady=5, ipadx=50, ipady=50, sticky="nsew")

            month_name = calendar.month_name[month]
            rent_info_list = self.monthly_rents.get((current_year, month), [])
            if rent_info_list:
                text_rent = ""
                for idx, (monto, day_paid, persona) in enumerate(rent_info_list, start=1):
                    text_rent += f"\nRenta#{idx}: ${monto:.2f}, pagado el {day_paid}, persona: {persona}"
            else:
                text_rent = "\nSin renta registrada"

            lbl_text = f"{month_name} {current_year}{text_rent}"
            lbl_m = tb.Label(frame_month, text=lbl_text, font=("Segoe UI",10), justify="left")
            lbl_m.pack(padx=5, pady=5)

            if self.current_user_role in ("admin","encargado"):
                btn_det = tb.Button(frame_month, text="Detalles", command=lambda m=month: self.open_month_dialog(m))
                btn_det.pack(pady=5)

    def open_month_dialog(self, month):
        if self.current_user_role not in ("admin","encargado"):
            messagebox.showerror("Permiso denegado","No puedes registrar rentas.")
            return
        current_year = datetime.date.today().year
        rent_info_list = self.monthly_rents.get((current_year, month), [])
        month_name = calendar.month_name[month]
        if not rent_info_list:
            msg=f"No hay renta registrada para {month_name} {current_year}.\n¿Registrar?"
        else:
            detail_rents="\n".join([f"Renta#{i+1}: ${r[0]:.2f}, pagado el {r[1]}, persona: {r[2]}" for i,r in enumerate(rent_info_list)])
            msg=f"Rentas actuales para {month_name} {current_year}:\n{detail_rents}\n\n¿Agregar más rentas?"
        update=messagebox.askyesno("Renta Mensual", msg)
        if not update:
            return

        existing_count=len(rent_info_list)
        max_to_add=5-existing_count
        if max_to_add<=0:
            messagebox.showinfo("Renta Mensual","Ya hay 5 rentas.")
            return

        how_many=simpledialog.askinteger("Renta Mensual", f"¿Cuántas rentas desea agregar? (máx {max_to_add})", parent=self.root, minvalue=1, maxvalue=max_to_add)
        if not how_many:
            return

        for _ in range(how_many):
            rent_str=simpledialog.askstring("Registrar Renta", f"Monto de la renta en {month_name}/{current_year}:", parent=self.root)
            if not rent_str:
                continue
            try:
                rent=float(rent_str)
            except:
                messagebox.showerror("Error","Monto inválido.")
                continue
            persona=self.ask_for_limpieza_carnicero()
            if not persona:
                persona="N/A"
            day_paid=datetime.date.today().strftime("%d/%m/%Y")
            rent_info_list.append((rent, day_paid, persona))
            logging.info(f"Renta agregada para {month_name}/{current_year}: {rent}, persona={persona}")  # [MEJORA] LOG

        self.monthly_rents[(current_year, month)] = rent_info_list
        messagebox.showinfo("Renta Mensual", f"Renta(s) registrada(s) para {month_name}/{current_year}.", parent=self.root)
        self.show_months_view()

    def ask_for_limpieza_carnicero(self):
        c=self.db_manager.execute("SELECT username, role FROM users WHERE role IN ('limpieza','carnicero')")
        results=c.fetchall() if c else []
        if not results:
            return None
        top=tb.Toplevel(self.root)
        top.title("Seleccionar Persona")
        top.geometry("300x200")
        lb=tk.Listbox(top)
        lb.pack(fill="both", expand=True)
        for row in results:
            display=f"{row[0]} ({row[1]})"
            lb.insert("end", display)
        selected=[None]
        def on_ok():
            idx=lb.curselection()
            if not idx:
                top.destroy()
                return
            selected[0]=lb.get(idx[0])
            top.destroy()
        tb.Button(top, text="OK", command=on_ok, bootstyle="success").pack(pady=5)
        top.wait_window()
        return selected[0]

    def record_weekly_payment(self):
        if self.current_user_role not in ("admin","encargado"):
            messagebox.showerror("Permiso denegado","Solo admin/encargado registra pagos semanales.")
            return
        today=datetime.date.today()
        monday=today-datetime.timedelta(days=today.weekday())
        year,week_num,_=monday.isocalendar()
        pay_str=simpledialog.askstring("Registrar Pago Semanal", f"Monto pagado en semana {week_num} (año {year}):", parent=self.root)
        if pay_str:
            try:
                pay=float(pay_str)
                self.weekly_payments[(year,week_num)] = pay
                messagebox.showinfo("Pago Semanal", f"Pago registrado para semana {week_num}/{year}: ${pay:.2f}", parent=self.root)
                logging.info(f"Pago semanal registrado: semana {week_num}/{year}, monto={pay}")  # [MEJORA] LOG
            except:
                messagebox.showerror("Error","Monto inválido.")

    def record_monthly_rent(self):
        if self.current_user_role not in ("admin","encargado"):
            messagebox.showerror("Permiso denegado","Solo admin/encargado registra rentas mensuales.")
            return
        today=datetime.date.today()
        year=today.year
        month=today.month
        rent_info_list=self.monthly_rents.get((year,month),[])
        max_to_add=5-len(rent_info_list)
        if max_to_add<=0:
            messagebox.showinfo("Renta Mensual","Ya hay 5 rentas.")
            return

        how_many=simpledialog.askinteger("Renta Mensual", f"¿Cuántas rentas desea agregar? (máx {max_to_add})", parent=self.root, minvalue=1, maxvalue=max_to_add)
        if not how_many:
            return

        for _ in range(how_many):
            rent_str=simpledialog.askstring("Registrar Renta Mensual", f"Monto de la renta en {month}/{year}:", parent=self.root)
            if not rent_str:
                continue
            try:
                rent=float(rent_str)
            except:
                messagebox.showerror("Error","Monto inválido.")
                continue
            persona=self.ask_for_limpieza_carnicero()
            if not persona:
                persona="N/A"
            day_paid=today.strftime("%d/%m/%Y")
            rent_info_list.append((rent, day_paid, persona))
            logging.info(f"Renta agregada para {month}/{year}: {rent}, persona={persona}")  # [MEJORA] LOG

        self.monthly_rents[(year,month)] = rent_info_list
        messagebox.showinfo("Renta Mensual", f"Renta(s) registrada(s) para {month}/{year}.", parent=self.root)
        self.show_months_view()

    # ------------------ REPORTES ------------------
    def setup_reports_tab(self):
        for w in self.frame_reports.winfo_children():
            w.destroy()

        lbl_title=tb.Label(self.frame_reports, text="Reportes", font=("Segoe UI",16,"bold"))
        lbl_title.pack(pady=15)
        btn_frame=tb.Frame(self.frame_reports)
        btn_frame.pack(pady=10)

        tb.Button(btn_frame, text="Cerrar Caja", command=self.cerrar_caja, bootstyle="success").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Reporte Inventario", command=self.report_inventory, bootstyle="primary").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Reporte Ventas", command=self.report_sales, bootstyle="primary").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Reporte Gráfico", command=self.report_graph, bootstyle="primary").pack(side="left", padx=5)

    def cerrar_caja(self):
        fecha=datetime.date.today().strftime("%Y-%m-%d")
        hora_cierre=datetime.datetime.now().strftime("%H:%M:%S")
        total_surt = sum(g for _,g in self.surtidores_expenses)
        ganancia = self.total_amount - total_surt

        summary = f"Cierre de Caja - {fecha} {hora_cierre}\n"
        if self.login_time:
            summary += f"Hora de inicio de turno: {self.login_time.strftime('%H:%M:%S')}\n"
        summary += f"Cuenta Total: ${self.total_amount:.2f}\n"
        summary += f"Gasto en surtidores: ${total_surt:.2f}\n"
        summary += f"Ganancia del día: ${ganancia:.2f}\n"

        file_name = f"cierre_caja_{fecha}_{hora_cierre.replace(':','-')}.txt"
        try:
            with open(file_name,"w") as f:
                f.write(summary)
            messagebox.showinfo("Cerrar Caja", f"Cierre de caja guardado en {file_name}", parent=self.root)
            logging.info(f"Cierre de caja guardado: {file_name}")  # [MEJORA] LOG
        except Exception as e:
            messagebox.showerror("Error", f"No se pudo guardar el cierre de caja: {e}", parent=self.root)
            logging.error(f"No se pudo guardar cierre de caja: {e}")  # [MEJORA] LOG

    def report_inventory(self):
        win=tb.Toplevel(self.root)
        win.title("Reporte de Inventario")
        win.geometry("600x400")
        tree=tb.Treeview(win, columns=("ID","Código","Nombre","Categoría","Stock","Precio"), show="headings", style="Excel.Treeview")
        for c in ("ID","Código","Nombre","Categoría","Stock","Precio"):
            tree.heading(c, text=c)
            tree.column(c, width=80)
        tree.pack(fill="both", expand=True)
        cur=self.db_manager.execute("SELECT id, code, name, category, stock, price FROM products")
        for row in cur.fetchall():
            tree.insert("", "end", values=row)

    def report_sales(self):
        win=tb.Toplevel(self.root)
        win.title("Reporte de Ventas")
        win.geometry("600x400")
        txt=tk.Text(win, font=("Segoe UI",12))
        txt.pack(fill="both", expand=True)
        reporte="Ventas:\n"+"-"*40+"\n"
        for prod in self.account_items:
            reporte+=f"{prod['name']} - {prod['desc']} - {prod['cantidad']} x ${prod['precio_unit']:.2f} = ${prod['subtotal']:.2f}\n"
        reporte+=f"\nTotal Actual: ${self.total_amount:.2f}\n"
        txt.insert("1.0", reporte)

    def report_graph(self):
        cur=self.db_manager.execute("SELECT name, stock FROM products")
        datos=cur.fetchall()
        if not datos:
            messagebox.showinfo("Reporte Gráfico","No hay datos para mostrar.")
            return
        nombres=[d[0] for d in datos]
        stocks=[d[1] for d in datos]
        plt.figure(figsize=(8,5))
        plt.bar(nombres, stocks)  # sin color forzado
        plt.xlabel("Producto")
        plt.ylabel("Stock")
        plt.title("Stock de Inventario")
        plt.xticks(rotation=45, ha="right")
        plt.tight_layout()
        plt.show()

    # ------------------ USUARIOS ------------------
    def setup_users_tab(self):
        for w in self.frame_users.winfo_children():
            w.destroy()

        lbl_title=tb.Label(self.frame_users, text="Gestión de Usuarios", font=("Segoe UI",16,"bold"))
        lbl_title.pack(pady=15)

        columns=("ID","Usuario","Rol")
        self.tree_users=tb.Treeview(self.frame_users, columns=columns, show="headings", style="Excel.Treeview")
        for col in columns:
            self.tree_users.heading(col, text=col)
            self.tree_users.column(col, width=150)
        self.tree_users.pack(fill="both", expand=True, padx=10, pady=10)

        btn_frame=tb.Frame(self.frame_users)
        btn_frame.pack(pady=10)
        tb.Button(btn_frame, text="Agregar Usuario", command=self.add_user, bootstyle="success").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Editar Usuario", command=self.edit_user, bootstyle="warning").pack(side="left", padx=5)
        tb.Button(btn_frame, text="Eliminar Usuario", command=self.delete_user, bootstyle="danger").pack(side="left", padx=5)

        self.load_users()

    def load_users(self):
        for row in self.tree_users.get_children():
            self.tree_users.delete(row)
        cur=self.db_manager.execute("SELECT id, username, role FROM users")
        for r in cur.fetchall():
            self.tree_users.insert("", "end", values=r)

    def add_user(self):
        UserWindow(self, "Agregar Usuario")

    def edit_user(self):
        sel=self.tree_users.selection()
        if not sel:
            messagebox.showwarning("Editar Usuario","Seleccione un usuario para editar.")
            return
        data=self.tree_users.item(sel[0])["values"]
        UserWindow(self, "Editar Usuario", data)

    def delete_user(self):
        sel=self.tree_users.selection()
        if not sel:
            messagebox.showwarning("Eliminar Usuario","Seleccione un usuario para eliminar.")
            return
        data=self.tree_users.item(sel[0])["values"]
        if data[1]=="admin":
            messagebox.showerror("Error","No se puede eliminar al usuario admin.")
            return
        if messagebox.askyesno("Eliminar Usuario", f"¿Está seguro de eliminar al usuario {data[1]}?"):
            try:
                self.db_manager.execute("DELETE FROM users WHERE id=?", (data[0],))
                messagebox.showinfo("Eliminar Usuario","Usuario eliminado correctamente.")
                logging.info(f"Usuario eliminado: {data[1]} (ID: {data[0]})")  # [MEJORA] LOG
                self.load_users()
            except Exception as e:
                messagebox.showerror("Error", f"No se pudo eliminar el usuario: {e}")
                logging.error(f"No se pudo eliminar el usuario {data[1]}: {e}")  # [MEJORA] LOG

    # ------------------ CONFIGURACIÓN ------------------
    def setup_config_tab(self):
        for w in self.frame_config.winfo_children():
            w.destroy()

        lbl_title=tb.Label(self.frame_config, text="Configuración", font=("Segoe UI",16,"bold"))
        lbl_title.pack(pady=20)

        tb.Button(self.frame_config, text="Iniciar Escaneo", command=self.start_scan, bootstyle="primary").pack(pady=10)
        self.lbl_scan_result=tb.Label(self.frame_config, text="Resultado del escaneo:")
        self.lbl_scan_result.pack(pady=10)

        tb.Button(self.frame_config, text="Cambiar % Ganancia General", command=self.change_general_margin, bootstyle="info").pack(pady=5)
        tb.Button(self.frame_config, text="Cambiar % Ganancia Frutas/Verduras", command=self.change_fv_margin, bootstyle="info").pack(pady=5)

    def change_general_margin(self):
        margin_str=simpledialog.askstring("Cambiar % Ganancia General","Ingresa el factor (ej. 1.3 para 30%)", parent=self.root)
        if not margin_str:
            return
        try:
            val=float(margin_str)
            if val<1.0:
                messagebox.showerror("Error","El factor debe ser >=1.0",parent=self.root)
                return
            self.general_margin=val
            messagebox.showinfo("Margen General", f"Se cambió a {val}", parent=self.root)
            logging.info(f"Margen general cambiado a {val}")  # [MEJORA] LOG
        except:
            messagebox.showerror("Error","Valor inválido.",parent=self.root)

    def change_fv_margin(self):
        margin_str=simpledialog.askstring("Cambiar % Ganancia Frutas/Verduras","Ingresa el factor (ej. 1.3 para 30%)", parent=self.root)
        if not margin_str:
            return
        try:
            val=float(margin_str)
            if val<1.0:
                messagebox.showerror("Error","El factor debe ser >=1.0",parent=self.root)
                return
            self.fv_margin=val
            messagebox.showinfo("Margen Frutas/Verduras", f"Se cambió a {val}", parent=self.root)
            logging.info(f"Margen frutas/verduras cambiado a {val}")  # [MEJORA] LOG
        except:
            messagebox.showerror("Error","Valor inválido.",parent=self.root)

    def start_scan(self):
        code=self.real_scan()
        if code:
            c=self.db_manager.execute("SELECT * FROM products WHERE code=?", (code,))
            product=c.fetchone()
            if product:
                result_text=(f"Código: {product[1]}\n"
                             f"Nombre: {product[2]}\n"
                             f"Categoría: {product[3]}\n"
                             f"Stock: {product[4]}\n"
                             f"Precio: {product[5]}")
                self.lbl_scan_result.config(text=result_text)
            else:
                self.lbl_scan_result.config(text="Producto no encontrado.")
        else:
            self.lbl_scan_result.config(text="No se detectó ningún código.")

    def real_scan(self):
        cap=cv2.VideoCapture(0)
        detected_code=None
        while True:
            ret, frame=cap.read()
            if not ret:
                break
            barcodes=decode(frame)
            for bc in barcodes:
                detected_code=bc.data.decode("utf-8")
                (x,y,w,h)=bc.rect
                cv2.rectangle(frame,(x,y),(x+w,y+h),(0,255,0),2)
                cv2.putText(frame, detected_code,(x,y-10),cv2.FONT_HERSHEY_SIMPLEX,0.5,(0,255,0),2)
                break
            cv2.imshow("Escanear - Presiona Q para salir", frame)
            if detected_code is not None:
                break
            if cv2.waitKey(1)&0xFF==ord("q"):
                break
        cap.release()
        cv2.destroyAllWindows()
        return detected_code

# ------------------ CLASES SECUNDARIAS ------------------
class AddProductWindow:
    def __init__(self, app):
        self.app=app
        self.window=tb.Toplevel(app.root)
        self.window.title("Agregar Producto")
        self.window.geometry("400x300")
        self.create_widgets()

    def create_widgets(self):
        frame=tb.Frame(self.window, padding=10)
        frame.pack(fill="both", expand=True)

        tb.Label(frame, text="Código:").grid(row=0, column=0, sticky="e", pady=5)
        self.entry_code=tb.Entry(frame)
        self.entry_code.grid(row=0, column=1, pady=5)

        tb.Label(frame, text="Nombre:").grid(row=1, column=0, sticky="e", pady=5)
        self.entry_name=tb.Entry(frame)
        self.entry_name.grid(row=1, column=1, pady=5)

        tb.Label(frame, text="Categoría:").grid(row=2, column=0, sticky="e", pady=5)
        self.entry_category=tb.Entry(frame)
        self.entry_category.grid(row=2, column=1, pady=5)

        tb.Label(frame, text="Stock:").grid(row=3, column=0, sticky="e", pady=5)
        self.entry_stock=tb.Entry(frame)
        self.entry_stock.grid(row=3, column=1, pady=5)

        tb.Label(frame, text="Costo:").grid(row=4, column=0, sticky="e", pady=5)
        self.entry_price=tb.Entry(frame)
        self.entry_price.grid(row=4, column=1, pady=5)

        tb.Label(frame, text="(Se aplicará el margen según categoría)").grid(row=5, column=0, columnspan=2)

        tb.Button(frame, text="Agregar", command=self.add_product, bootstyle="success").grid(row=6, column=0, columnspan=2, pady=10)

    def add_product(self):
        code=self.entry_code.get().strip()
        name=self.entry_name.get().strip()
        category=self.entry_category.get().strip().capitalize()
        try:
            stock=int(self.entry_stock.get().strip())
        except:
            stock=0
        try:
            cost=float(self.entry_price.get().strip())
        except:
            cost=0.0

        if category in ("Fruta","Verdura"):
            final_price=custom_round(cost*self.app.fv_margin)
        else:
            final_price=custom_round(cost*self.app.general_margin)

        # [MEJORA] Validación adicional
        if not code or not name:
            messagebox.showerror("Error","Código y nombre son obligatorios.", parent=self.window)
            return

        try:
            self.app.db_manager.execute(
                "INSERT INTO products (code,name,category,stock,price,image_path) VALUES (?,?,?,?,?,?)",
                (code,name,category,stock,final_price,"")
            )
            messagebox.showinfo("Agregar Producto","Producto agregado correctamente.",parent=self.window)
            logging.info(f"Producto agregado: {name} (Cod:{code}, Cat:{category}, Precio:{final_price})")  # [MEJORA] LOG
            self.window.destroy()
        except sqlite3.IntegrityError:
            messagebox.showerror("Error","El código ya existe en la base de datos.",parent=self.window)
            logging.warning(f"Intento de agregar producto con código duplicado: {code}")  # [MEJORA] LOG
        except Exception as e:
            messagebox.showerror("Error",f"No se pudo agregar el producto: {e}",parent=self.window)
            logging.error(f"No se pudo agregar producto {name}: {e}")  # [MEJORA] LOG

class EditProductWindow:
    def __init__(self, app, data):
        self.app=app
        self.product_id=data[0]
        self.data=data
        self.window=tb.Toplevel(app.root)
        self.window.title("Editar Producto")
        self.window.geometry("400x300")
        self.create_widgets()

    def create_widgets(self,):
        frame=tb.Frame(self.window, padding=10)
        frame.pack(fill="both", expand=True)

        tb.Label(frame, text="Código:").grid(row=0, column=0, sticky="e", pady=5)
        self.entry_code=tb.Entry(frame)
        self.entry_code.insert(0,self.data[1])
        self.entry_code.grid(row=0, column=1, pady=5)

        tb.Label(frame, text="Nombre:").grid(row=1, column=0, sticky="e", pady=5)
        self.entry_name=tb.Entry(frame)
        self.entry_name.insert(0,self.data[2])
        self.entry_name.grid(row=1, column=1, pady=5)

        tb.Label(frame, text="Categoría:").grid(row=2, column=0, sticky="e", pady=5)
        self.entry_category=tb.Entry(frame)
        self.entry_category.insert(0,self.data[3])
        self.entry_category.grid(row=2, column=1, pady=5)

        tb.Label(frame, text="Stock:").grid(row=3, column=0, sticky="e", pady=5)
        self.entry_stock=tb.Entry(frame)
        self.entry_stock.insert(0,self.data[4])
        self.entry_stock.grid(row=3, column=1, pady=5)

        tb.Label(frame, text="Costo:").grid(row=4, column=0, sticky="e", pady=5)
        cat=self.data[3].capitalize()
        if cat in ("Fruta","Verdura"):
            cost=self.data[5]/self.app.fv_margin
        else:
            cost=self.data[5]/self.app.general_margin

        self.entry_cost=tb.Entry(frame)
        self.entry_cost.insert(0,f"{cost:.2f}")
        self.entry_cost.grid(row=4, column=1, pady=5)

        tb.Button(frame, text="Actualizar", command=self.update_product, bootstyle="success").grid(row=5, column=0, columnspan=2, pady=10)

    def update_product(self):
        code=self.entry_code.get().strip()
        name=self.entry_name.get().strip()
        category=self.entry_category.get().strip().capitalize()
        try:
            stock=int(self.entry_stock.get().strip())
        except:
            stock=0
        try:
            cost=float(self.entry_cost.get().strip())
        except:
            cost=0.0

        if category in ("Fruta","Verdura"):
            final_price=custom_round(cost*self.app.fv_margin)
        else:
            final_price=custom_round(cost*self.app.general_margin)

        if not code or not name:
            messagebox.showerror("Error","Código y nombre son obligatorios.", parent=self.window)
            return

        try:
            self.app.db_manager.execute(
                "UPDATE products SET code=?, name=?, category=?, stock=?, price=? WHERE id=?",
                (code,name,category,stock,final_price,self.product_id)
            )
            messagebox.showinfo("Editar Producto","Producto actualizado correctamente.",parent=self.window)
            logging.info(f"Producto actualizado: {name} (ID:{self.product_id}, Precio:{final_price})")  # [MEJORA] LOG
            self.window.destroy()
        except sqlite3.IntegrityError:
            messagebox.showerror("Error","El código ya existe en la base de datos.",parent=self.window)
            logging.warning(f"Intento de actualizar producto con código duplicado: {code}")  # [MEJORA] LOG
        except Exception as e:
            messagebox.showerror("Error",f"No se pudo actualizar el producto: {e}",parent=self.window)
            logging.error(f"No se pudo actualizar producto {name} (ID:{self.product_id}): {e}")  # [MEJORA] LOG

class UserWindow:
    def __init__(self, app, title, data=None):
        self.app=app
        self.window=tb.Toplevel(app.root)
        self.window.title(title)
        self.window.geometry("500x300")
        self.data=data
        self.create_widgets()

    def create_widgets(self):
        frame=tb.Frame(self.window, padding=10)
        frame.pack(fill="both", expand=True)

        tb.Label(frame, text="Usuario:").grid(row=0, column=0, sticky="e", pady=5)
        self.entry_user=tb.Entry(frame)
        self.entry_user.grid(row=0, column=1, pady=5)

        tb.Label(frame, text="Contraseña:").grid(row=1, column=0, sticky="e", pady=5)
        self.entry_pass=tb.Entry(frame, show="*")
        self.entry_pass.grid(row=1, column=1, pady=5)

        tb.Label(frame, text="Rol:").grid(row=2, column=0, sticky="e", pady=5)
        self.entry_role=tb.Entry(frame)
        self.entry_role.grid(row=2, column=1, pady=5)

        if self.data:
            self.entry_user.insert(0,self.data[1])
            self.entry_role.insert(0,self.data[2])
            btn_text="Actualizar"
        else:
            btn_text="Agregar"

        tb.Button(frame, text=btn_text, command=self.save_user, bootstyle="success").grid(row=3, column=0, columnspan=2, pady=10)

    def save_user(self):
        user=self.entry_user.get().strip()
        pwd=self.entry_pass.get().strip()
        role=self.entry_role.get().strip()
        if not user or not pwd:
            messagebox.showerror("Error","Usuario y contraseña obligatorios.", parent=self.window)
            return

        # [MEJORA] En caso de contraseñas hasheadas
        if USE_HASHED_PASSWORDS:
            pw_stored = hash_password(pwd)
        else:
            pw_stored = pwd

        try:
            if self.data:
                self.app.db_manager.execute("UPDATE users SET username=?, password=?, role=? WHERE id=?",
                                            (user,pw_stored,role,self.data[0]))
                logging.info(f"Usuario actualizado: {user} (ID:{self.data[0]})")  # [MEJORA] LOG
            else:
                self.app.db_manager.execute("INSERT INTO users (username,password,role) VALUES (?,?,?)",
                                            (user,pw_stored,role))
                logging.info(f"Usuario agregado: {user}")  # [MEJORA] LOG
            messagebox.showinfo("Usuarios","Operación realizada correctamente.", parent=self.window)
            self.app.load_users()
            self.window.destroy()
        except sqlite3.IntegrityError:
            messagebox.showerror("Error","Ese usuario ya existe.",parent=self.window)
            logging.warning(f"Intento de crear/actualizar usuario repetido: {user}")  # [MEJORA] LOG
        except Exception as e:
            messagebox.showerror("Error",f"No se pudo completar la operación: {e}",parent=self.window)
            logging.error(f"No se pudo crear/actualizar usuario {user}: {e}")  # [MEJORA] LOG

# ------------------ MAIN ------------------
if __name__=="__main__":
    # Usa un tema válido, ej. "cosmo"
    root = tb.Window(themename="cosmo")
    root.state("zoomed")
    db_manager = DatabaseManager()
    app = BarcodeApp(root, db_manager)
    show_login(root, db_manager, app)
    root.mainloop()

