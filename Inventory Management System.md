import tkinter as tk
from tkinter import messagebox, ttk
import sqlite3

# I don't fully understand sqlite but this worked so I kept it
db_link = sqlite3.connect('inventory_data.db')
db_ops = db_link.cursor()

# create login table if not already
db_ops.execute('''
CREATE TABLE IF NOT EXISTS users (
    username TEXT PRIMARY KEY,
    password TEXT NOT NULL
)
''')

# table for storing items (again)
db_ops.execute('''
CREATE TABLE IF NOT EXISTS items (
    product_id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    quantity INTEGER NOT NULL,
    price REAL NOT NULL
)
''')

# added this so I don't get locked out
db_ops.execute("INSERT OR IGNORE INTO users VALUES (?, ?)", ("python", "python123"))
db_link.commit()


# --- login screen because sir asked for it ---
class LoginBox:
    def __init__(self, screen):
        self.screen = screen
        self.screen.title("Login (pls enter something)")
        self.screen.geometry("300x200")

        tk.Label(screen, text="Username:").pack(pady=5)
        self.userTyped = tk.Entry(screen)
        self.userTyped.pack()

        tk.Label(screen, text="Password:").pack(pady=5)
        self.passTyped = tk.Entry(screen, show="*")
        self.passTyped.pack()

        tk.Button(screen, text="Login", command=self.validateMe).pack(pady=15)

    def validateMe(self):
        gotUser = self.userTyped.get()
        gotPass = self.passTyped.get()

        db_ops.execute("SELECT * FROM users WHERE username=? AND password=?", (gotUser, gotPass))
        isHere = db_ops.fetchone()

        if isHere:
            print("yay login success")
            self.screen.destroy()
            StockApp()
        else:
            messagebox.showerror("Try Again", "Wrong login info maybe? Idk")


# --- the actual app thing ---
class StockApp:
    def __init__(self):
        self.win = tk.Tk()
        self.win.title("My Inventory (intern ver)")
        self.win.geometry("750x500")

        self.drawEverything()
        self.showData()

        self.win.mainloop()

    def drawEverything(self):
        # table to show stuff
        self.listArea = ttk.Treeview(self.win, columns=("ID", "Item", "Qty", "Rs"), show="headings")
        self.listArea.heading("ID", text="ID")
        self.listArea.heading("Item", text="Name")
        self.listArea.heading("Qty", text="Qty")
        self.listArea.heading("Rs", text="Price")
        self.listArea.pack(fill=tk.BOTH, expand=True)

        self.listArea.bind('<ButtonRelease-1>', self.whenIClick)

        # form for entry
        box = tk.Frame(self.win)
        box.pack(pady=10)

        tk.Label(box, text="Item").grid(row=0, column=0)
        tk.Label(box, text="Qty").grid(row=0, column=1)
        tk.Label(box, text="Price").grid(row=0, column=2)

        self.whatItem = tk.Entry(box)
        self.whatItem.grid(row=1, column=0)

        self.howMany = tk.Entry(box)
        self.howMany.grid(row=1, column=1)

        self.costPerOne = tk.Entry(box)
        self.costPerOne.grid(row=1, column=2)

        # buttons which do the real work
        tk.Button(box, text="Add", command=self.addToList).grid(row=1, column=3, padx=5)
        tk.Button(box, text="Edit", command=self.editItem).grid(row=1, column=4, padx=5)
        tk.Button(box, text="Del", command=self.deleteThis).grid(row=1, column=5, padx=5)
        tk.Button(box, text="Check Stock", command=self.showLessStock).grid(row=1, column=6, padx=5)

    def showData(self):
        # deleting stuff just in case
        self.listArea.delete(*self.listArea.get_children())
        db_ops.execute("SELECT * FROM items")
        allRows = db_ops.fetchall()

        for eachRow in allRows:
            self.listArea.insert('', tk.END, values=eachRow)

    def addToList(self):
        name = self.whatItem.get()
        qty = self.howMany.get()
        cost = self.costPerOne.get()

        # trying int conversion like sir said
        try:
            qty_num = int(qty)
            price_num = float(cost)
        except:
            print("bad input detected")
            messagebox.showinfo("Oops", "Please enter numbers for qty and price.")
            return

        if name == "":
            messagebox.showwarning("Blank", "Name can't be empty bro.")
            return

        db_ops.execute("INSERT INTO items (name, quantity, price) VALUES (?, ?, ?)", (name, qty_num, price_num))
        db_link.commit()
        self.showData()
        print("Added:", name)

    def whenIClick(self, e):
        chosen = self.listArea.focus()
        details = self.listArea.item(chosen, "values")
        if details:
            self.whatItem.delete(0, tk.END)
            self.howMany.delete(0, tk.END)
            self.costPerOne.delete(0, tk.END)

            self.whatItem.insert(0, details[1])
            self.howMany.insert(0, details[2])
            self.costPerOne.insert(0, details[3])

    def editItem(self):
        selected = self.listArea.focus()
        values = self.listArea.item(selected, "values")
        if not values:
            print("nothing selected to update")
            return

        updatedName = self.whatItem.get()
        try:
            updatedQty = int(self.howMany.get())
            updatedPrice = float(self.costPerOne.get())
        except:
            messagebox.showwarning("Err", "Numbers are not looking good.")
            return

        db_ops.execute("UPDATE items SET name=?, quantity=?, price=? WHERE product_id=?",
                       (updatedName, updatedQty, updatedPrice, values[0]))
        db_link.commit()
        self.showData()

    def deleteThis(self):
        selected = self.listArea.focus()
        row = self.listArea.item(selected, "values")
        if not row:
            return
        db_ops.execute("DELETE FROM items WHERE product_id=?", (row[0],))
        db_link.commit()
        self.showData()

    def showLessStock(self):
        db_ops.execute("SELECT * FROM items WHERE quantity < 5")
        lowList = db_ops.fetchall()

        if not lowList:
            messagebox.showinfo("All Good", "No item is that low (yet).")
        else:
            msg = "Items with low stock:\n\n"
            for x in lowList:
                msg += f"{x[1]} - Qty: {x[2]}\n"
            messagebox.showwarning("Restock pls!", msg)


# --- finally, start it all ---
if __name__ == "__main__":
    launchApp = tk.Tk()
    LoginBox(launchApp)
    launchApp.mainloop()
