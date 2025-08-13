import sqlite3
from tkinter import *
from tkinter import messagebox, ttk

# ---------- Database ----------
conn = sqlite3.connect("students.db")
cur = conn.cursor()
cur.execute("""
CREATE TABLE IF NOT EXISTS students (
    roll_no INTEGER PRIMARY KEY,
    name TEXT NOT NULL,
    marks INTEGER NOT NULL
)
""")
conn.commit()

# ---------- Helpers ----------
def sanitize_inputs():
    roll = roll_no_var.get().strip()
    name = name_var.get().strip()
    marks = marks_var.get().strip()
    return roll, name, marks

def load_selected_into_form(event=None):
    selected = student_table.focus()
    if not selected:
        return
    r, n, m = student_table.item(selected, "values")
    roll_no_var.set(r)
    name_var.set(n)
    marks_var.set(m)

def clear_fields():
    roll_no_var.set("")
    name_var.set("")
    marks_var.set("")

def show_all():
    cur.execute("SELECT * FROM students ORDER BY roll_no")
    rows = cur.fetchall()
    student_table.delete(*student_table.get_children())
    for row in rows:
        student_table.insert("", END, values=row)

# ---------- CRUD ----------
def add_student():
    roll, name, marks = sanitize_inputs()

    if not roll or not name or not marks:
        messagebox.showerror("Error", "Please fill Roll No, Name, and Marks.")
        return

    # validate numbers
    try:
        roll_i = int(roll)
        marks_i = int(marks)
        if not (0 <= marks_i <= 100):
            raise ValueError
    except ValueError:
        messagebox.showerror("Error", "Roll No must be a number.\nMarks must be a number 0–100.")
        return

    try:
        cur.execute("INSERT INTO students (roll_no, name, marks) VALUES (?, ?, ?)", (roll_i, name, marks_i))
        conn.commit()
        show_all()
        clear_fields()
        messagebox.showinfo("Success", "Student added.")
    except sqlite3.IntegrityError:
        messagebox.showerror("Error", "That Roll No already exists.")

def update_student():
    roll, name, marks = sanitize_inputs()
    if not roll:
        messagebox.showerror("Error", "Enter the Roll No to update.")
        return
    try:
        roll_i = int(roll)
    except ValueError:
        messagebox.showerror("Error", "Roll No must be a number.")
        return

    # If a field is empty, keep old value
    cur.execute("SELECT name, marks FROM students WHERE roll_no=?", (roll_i,))
    row = cur.fetchone()
    if not row:
        messagebox.showerror("Error", "No student found with that Roll No.")
        return

    current_name, current_marks = row
    new_name = name if name else current_name

    try:
        new_marks = int(marks) if marks else int(current_marks)
        if not (0 <= new_marks <= 100):
            raise ValueError
    except ValueError:
        messagebox.showerror("Error", "Marks must be a number 0–100.")
        return

    cur.execute("UPDATE students SET name=?, marks=? WHERE roll_no=?", (new_name, new_marks, roll_i))
    conn.commit()
    show_all()
    messagebox.showinfo("Success", "Student updated.")

def delete_student():
    # prefer selected row; else use roll_no field
    selected = student_table.focus()
    target_roll = None
    if selected:
        target_roll = student_table.item(selected, "values")[0]
    else:
        r = roll_no_var.get().strip()
        if r:
            target_roll = r

    if not target_roll:
        messagebox.showerror("Error", "Select a row or enter a Roll No to delete.")
        return

    if not messagebox.askyesno("Confirm", f"Delete student with Roll No {target_roll}?"):
        return

    cur.execute("DELETE FROM students WHERE roll_no=?", (target_roll,))
    conn.commit()
    show_all()
    clear_fields()
    messagebox.showinfo("Deleted", "Student deleted.")

def search_student():
    text = roll_no_var.get().strip() or name_var.get().strip()
    if not text:
        messagebox.showerror("Error", "Type Roll No or Name to search.")
        return
    # if number, search by roll_no; otherwise search by name (contains)
    try:
        rn = int(text)
        cur.execute("SELECT * FROM students WHERE roll_no=?", (rn,))
    except ValueError:
        cur.execute("SELECT * FROM students WHERE name LIKE ?", (f"%{text}%",))
    rows = cur.fetchall()
    student_table.delete(*student_table.get_children())
    for row in rows:
        student_table.insert("", END, values=row)

def add_demo_data():
    demo = [
        (1, "Aarav", 91),
        (2, "Isha", 84),
        (3, "Rohit", 76),
        (4, "Zara", 88),
        (5, "Vikram", 67),
    ]
    for r, n, m in demo:
        try:
            cur.execute("INSERT INTO students (roll_no, name, marks) VALUES (?, ?, ?)", (r, n, m))
        except sqlite3.IntegrityError:
            pass
    conn.commit()
    show_all()

# ---------- GUI ----------
root = Tk()
root.title("Student Result Management System")
root.geometry("860x560")

title = Label(root, text="Student Result Management System", font=("Segoe UI", 18, "bold"), bg="#cfe8ff")
title.pack(side=TOP, fill=X, pady=(0, 6))

form = Frame(root)
form.pack(padx=10, pady=6, fill=X)

roll_no_var = StringVar()
name_var = StringVar()
marks_var = StringVar()

Label(form, text="Roll No").grid(row=0, column=0, sticky="w", padx=6, pady=6)
Entry(form, textvariable=roll_no_var, width=15).grid(row=0, column=1, padx=6, pady=6)

Label(form, text="Name").grid(row=0, column=2, sticky="w", padx=6, pady=6)
Entry(form, textvariable=name_var, width=24).grid(row=0, column=3, padx=6, pady=6)

Label(form, text="Marks").grid(row=0, column=4, sticky="w", padx=6, pady=6)
Entry(form, textvariable=marks_var, width=10).grid(row=0, column=5, padx=6, pady=6)

btns = Frame(root)
btns.pack(padx=10, pady=4, fill=X)

Button(btns, text="Add", width=10, command=add_student).pack(side=LEFT, padx=4)
Button(btns, text="Update", width=10, command=update_student).pack(side=LEFT, padx=4)
Button(btns, text="Delete", width=10, command=delete_student).pack(side=LEFT, padx=4)
Button(btns, text="Search", width=10, command=search_student).pack(side=LEFT, padx=4)
Button(btns, text="Show All", width=10, command=show_all).pack(side=LEFT, padx=4)
Button(btns, text="Clear", width=10, command=clear_fields).pack(side=LEFT, padx=4)
Button(btns, text="Demo Data", width=12, command=add_demo_data).pack(side=LEFT, padx=12)

table_frame = Frame(root)
table_frame.pack(padx=10, pady=8, fill=BOTH, expand=True)

scroll_y = Scrollbar(table_frame, orient=VERTICAL)
scroll_x = Scrollbar(table_frame, orient=HORIZONTAL)

student_table = ttk.Treeview(
    table_frame,
    columns=("roll", "name", "marks"),
    yscrollcommand=scroll_y.set,
    xscrollcommand=scroll_x.set
)

scroll_y.pack(side=RIGHT, fill=Y)
scroll_x.pack(side=BOTTOM, fill=X)
scroll_y.config(command=student_table.yview)
scroll_x.config(command=student_table.xview)

student_table.heading("roll", text="Roll No")
student_table.heading("name", text="Name")
student_table.heading("marks", text="Marks")
student_table["show"] = "headings"
student_table.column("roll", width=120, anchor=CENTER)
student_table.column("name", width=340)
student_table.column("marks", width=120, anchor=CENTER)
student_table.pack(fill=BOTH, expand=True)

student_table.bind("<<TreeviewSelect>>", load_selected_into_form)

show_all()
root.mainloop()
