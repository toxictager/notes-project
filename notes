import tkinter as tk
from tkinter import ttk, messagebox, simpledialog, Menu
import os
import json
from tkinter.scrolledtext import ScrolledText
import requests
import sys
import subprocess


CURRENT_VERSION = "1.0.2"  # Change this when you build a new version
VERSION_URL = "https://raw.githubusercontent.com/toxictager/myapp-updates/refs/heads/main/version.txt"
UPDATE_URL = "https://github.com/toxictager/myapp-updates/raw/main/notes.exe"

def check_for_update():
    try:
        latest_version = requests.get(VERSION_URL).text.strip()
        if latest_version != CURRENT_VERSION:
            answer = messagebox.askyesno("Update Available", f"A new version ({latest_version}) is available. Update now?")
            if answer:
                perform_update()
    except Exception as e:
        print(f"[Updater] Error checking for update: {e}")

def perform_update():
    try:
        current_path = os.path.abspath(sys.argv[0])
        temp_new_exe = current_path + ".new"
        with requests.get(UPDATE_URL, stream=True) as r:
            r.raise_for_status()
            with open(temp_new_exe, 'wb') as f:
                for chunk in r.iter_content(chunk_size=8192):
                    f.write(chunk)

        # Create a batch file to replace the old exe and relaunch the new one
        bat_script = f"""
        @echo off
        timeout /t 2 > nul
        del "{current_path}"
        ren "{temp_new_exe}" "{os.path.basename(current_path)}"
        start "" "{os.path.basename(current_path)}"
        del "%~f0"
        """
        bat_path = os.path.join(os.path.dirname(current_path), "update.bat")
        with open(bat_path, 'w') as f:
            f.write(bat_script)

        subprocess.Popen([bat_path], shell=True)
        sys.exit()

    except Exception as e:
        messagebox.showerror("Update Failed", f"Could not update the app:\n{e}")


# Define colors for light and dark mode
LIGHT_MODE = {
    'bg': 'white',
    'fg': 'black',
    'button_bg': 'lightgray',
    'button_fg': 'black',
    'tree_bg': 'white',
    'tree_fg': 'black',
    'tree_sel_bg': '#0078d7',
    'tree_sel_fg': 'white'
}

DARK_MODE = {
    'bg': '#2E2E2E',
    'fg': 'white',
    'button_bg': '#444444',
    'button_fg': 'white',
    'tree_bg': '#3C3F41',
    'tree_fg': 'white',
    'tree_sel_bg': '#0E639C',
    'tree_sel_fg': 'white'
}

CONFIG_FILE = "config.json"
NOTES_DIR = "notes"  # Directory to store notes

class NoteTakingApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Enhanced Note-Taking App")
        self.root.geometry("1000x700")
        
        # Initialize data
        self.note_structure = {}  # Only stores hierarchy, not content
        self.current_note_id = None
        self.currently_editing_path = None
        self.current_mode = self.load_theme()
        
        # Setup data directory
        self.setup_data_directory()
        
        # Setup UI
        self.setup_ui()
        
        # Load note structure
        self.load_note_structure()
        
        # Start autosave
        self.autosave_interval = 1000  # 1 second
        self.is_dirty = False
        self.autosave()
    
    def setup_data_directory(self):
        """Create the notes directory if it doesn't exist."""
        if not os.path.exists(NOTES_DIR):
            os.makedirs(NOTES_DIR)
    
    def setup_ui(self):
        """Setup the user interface with improved layout."""
        # Main PanedWindow for resizable panels
        self.main_pane = tk.PanedWindow(self.root, orient=tk.HORIZONTAL, bg=self.current_mode['bg'])
        self.main_pane.pack(fill=tk.BOTH, expand=True)
        
        # Left panel - Notes tree and buttons
        self.left_panel = tk.Frame(self.main_pane, bg=self.current_mode['bg'])
        self.main_pane.add(self.left_panel, width=300)
        
        # Button frame
        self.button_frame = tk.Frame(self.left_panel, bg=self.current_mode['bg'])
        self.button_frame.pack(fill=tk.X, padx=5, pady=5)
        
        # Buttons for note operations
        self.add_note_btn = tk.Button(self.button_frame, text="Add Note", 
                                     command=self.add_note,
                                     bg=self.current_mode['button_bg'], 
                                     fg=self.current_mode['button_fg'])
        self.add_note_btn.pack(side=tk.LEFT, expand=True, fill=tk.X, padx=2)
        
        self.add_subnote_btn = tk.Button(self.button_frame, text="Add Subnote", 
                                       command=self.add_subnote,
                                       bg=self.current_mode['button_bg'], 
                                       fg=self.current_mode['button_fg'])
        self.add_subnote_btn.pack(side=tk.LEFT, expand=True, fill=tk.X, padx=2)
        
        self.rename_btn = tk.Button(self.button_frame, text="Rename", 
                                   command=self.rename_note,
                                   bg=self.current_mode['button_bg'], 
                                   fg=self.current_mode['button_fg'])
        self.rename_btn.pack(side=tk.LEFT, expand=True, fill=tk.X, padx=2)
        
        self.delete_btn = tk.Button(self.button_frame, text="Delete", 
                                   command=self.delete_note,
                                   bg=self.current_mode['button_bg'], 
                                   fg=self.current_mode['button_fg'])
        self.delete_btn.pack(side=tk.LEFT, expand=True, fill=tk.X, padx=2)
        
        # Treeview for notes hierarchy
        self.notes_tree = ttk.Treeview(self.left_panel)
        self.notes_tree.pack(fill=tk.BOTH, expand=True, padx=5, pady=5)
        self.notes_tree.bind("<<TreeviewSelect>>", self.on_note_select)
        
        # Right-click context menu
        self.tree_menu = Menu(self.root, tearoff=0)
        self.tree_menu.add_command(label="Add Note", command=self.add_note)
        self.tree_menu.add_command(label="Add Subnote", command=self.add_subnote)
        self.tree_menu.add_command(label="Rename", command=self.rename_note)
        self.tree_menu.add_command(label="Delete", command=self.delete_note)
        
        # Bind right-click to show context menu
        self.notes_tree.bind("<Button-3>", self.show_context_menu)
        
        # Right panel - Note editor
        self.right_panel = tk.Frame(self.main_pane, bg=self.current_mode['bg'])
        self.main_pane.add(self.right_panel, width=700)
        
        # Note title
        self.note_title = tk.Label(self.right_panel, font=('Arial', 14, 'bold'), 
                                  bg=self.current_mode['bg'], fg=self.current_mode['fg'])
        self.note_title.pack(fill=tk.X, padx=10, pady=(10, 5))
        
        # Text editor with scrollbar
        self.text_editor = ScrolledText(self.right_panel, wrap=tk.WORD, 
                                       font=('Arial', 12), undo=True)
        self.text_editor.pack(fill=tk.BOTH, expand=True, padx=10, pady=5)
        self.text_editor.bind("<KeyRelease>", self.mark_dirty)
        
        # Status bar
        self.status_var = tk.StringVar()
        self.status_var.set("Ready")
        self.status_bar = tk.Label(self.root, textvariable=self.status_var, 
                                 relief=tk.SUNKEN, anchor="w",
                                 bg=self.current_mode['bg'], fg=self.current_mode['fg'])
        self.status_bar.pack(side=tk.BOTTOM, fill=tk.X)
        
        # Control buttons
        self.control_frame = tk.Frame(self.root, bg=self.current_mode['bg'])
        self.control_frame.pack(fill=tk.X)
        
        self.toggle_mode_btn = tk.Button(self.control_frame, text="Toggle Dark Mode", 
                                        command=self.toggle_dark_mode,
                                        bg=self.current_mode['button_bg'], 
                                        fg=self.current_mode['button_fg'])
        self.toggle_mode_btn.pack(side=tk.RIGHT, padx=5, pady=5)
        
        self.search_btn = tk.Button(self.control_frame, text="Search Notes", 
                                  command=self.search_notes,
                                  bg=self.current_mode['button_bg'], 
                                  fg=self.current_mode['button_fg'])
        self.search_btn.pack(side=tk.LEFT, padx=5, pady=5)
        
        # Apply theme
        self.apply_theme()
    
    def show_context_menu(self, event):
        """Show context menu on right-click."""
        item = self.notes_tree.identify_row(event.y)
        if item:
            self.notes_tree.selection_set(item)
            try:
                self.tree_menu.tk_popup(event.x_root, event.y_root)
            finally:
                self.tree_menu.grab_release()
    
    def load_note_structure(self):
        """Load note structure from the structure file."""
        structure_file = os.path.join(NOTES_DIR, "structure.json")
        
        if os.path.exists(structure_file):
            with open(structure_file, "r") as f:
                self.note_structure = json.load(f)
        else:
            # Create a default structure
            self.note_structure = {
                "Welcome": {
                    "children": {}
                }
            }
            # Create a welcome note file
            self.save_note_content("Welcome", "Welcome to your note-taking app!\n\nCreate new notes using the right-click menu or the buttons above.")
            self.save_note_structure()
        
        self.populate_tree()
    
    def save_note_structure(self):
        """Save the note structure to file."""
        structure_file = os.path.join(NOTES_DIR, "structure.json")
        with open(structure_file, "w") as f:
            json.dump(self.note_structure, f, indent=4)
    
    def get_note_filename(self, path):
        """Generate a unique filename for a note based on its path."""
        # Create a safe filename by joining path elements with double underscore
        filename = "__".join(path) + ".txt"
        return os.path.join(NOTES_DIR, filename)
    
    def save_note_content(self, note_name, content, parent_path=None):
        """Save a note's content to its corresponding file."""
        if parent_path is None:
            parent_path = []
        
        note_path = parent_path + [note_name]
        filename = self.get_note_filename(note_path)
        
        with open(filename, "w", encoding="utf-8") as f:
            f.write(content)
    
    def load_note_content(self, path):
        """Load a note's content from its file."""
        filename = self.get_note_filename(path)
        
        if os.path.exists(filename):
            with open(filename, "r", encoding="utf-8") as f:
                return f.read()
        return ""  # Return empty string if file doesn't exist
    
    def populate_tree(self):
        """Populate the treeview with notes."""
        self.notes_tree.delete(*self.notes_tree.get_children())
        self.add_tree_nodes("", self.note_structure)
    
    def add_tree_nodes(self, parent, nodes):
        """Recursively add nodes to the treeview."""
        for name, data in nodes.items():
            node_id = self.notes_tree.insert(parent, "end", text=name)
            if isinstance(data, dict) and "children" in data:
                self.add_tree_nodes(node_id, data["children"])
    
    def get_note_path(self, node_id):
        """Get the path to a note in the tree."""
        path = []
        while node_id:
            path.append(self.notes_tree.item(node_id)["text"])
            node_id = self.notes_tree.parent(node_id)
        return list(reversed(path))
    
    def get_note_by_path(self, path):
        """Get a note's structure by its path."""
        current = self.note_structure
        for part in path:
            if part in current:
                current = current[part]
            else:
                return None
        return current
    
    def on_note_select(self, event):
        """Handle note selection in the tree."""
        selected = self.notes_tree.selection()
        if not selected:
            return
        
        # Save any changes to the previously selected note first
        self.save_current_note()
        
        # Load the newly selected note
        node_id = selected[0]
        path = self.get_note_path(node_id)
        
        # Load note content from file
        content = self.load_note_content(path)
        
        self.current_note_id = node_id
        self.currently_editing_path = path
        self.note_title.config(text=" → ".join(path))
        self.text_editor.delete(1.0, tk.END)
        self.text_editor.insert(tk.END, content)
        self.is_dirty = False
        self.status_var.set("Note loaded")
    
    def save_current_note(self):
        """Save the current note content if it exists."""
        if self.currently_editing_path and self.is_dirty:
            content = self.text_editor.get(1.0, tk.END).strip()
            self.save_note_content(self.currently_editing_path[-1], content, 
                                  self.currently_editing_path[:-1])
            self.is_dirty = False
            self.status_var.set("Note saved")
    
    def add_note(self):
        """Add a new note."""
        selected = self.notes_tree.selection()
        parent_id = selected[0] if selected else ""
        
        note_name = simpledialog.askstring("New Note", "Enter note name:")
        if not note_name:
            return
        
        # Save current note content before adding new one
        self.save_current_note()
        
        if parent_id:
            # Add as a child of selected note
            parent_path = self.get_note_path(parent_id)
            parent = self.get_note_by_path(parent_path)
            
            # Check if note with same name exists
            if note_name in parent.get("children", {}):
                messagebox.showerror("Error", "A note with this name already exists here!")
                return
            
            # Ensure parent has children dict
            if not isinstance(parent, dict):
                parent = {"children": {}}
                self.set_note_at_path(parent_path, parent)
            
            if "children" not in parent:
                parent["children"] = {}
            
            # Add the new note
            parent["children"][note_name] = {"children": {}}
            
            # Save empty content for new note
            new_path = parent_path + [note_name]
            self.save_note_content(note_name, "", parent_path)
        else:
            # Add as a root note
            if note_name in self.note_structure:
                messagebox.showerror("Error", "A note with this name already exists!")
                return
            
            # Add to root structure
            self.note_structure[note_name] = {"children": {}}
            
            # Save empty content for new note
            self.save_note_content(note_name, "")
        
        # Save the updated structure
        self.save_note_structure()
        self.populate_tree()
        
        # Select the new note
        self.select_note_by_name(note_name, parent_id)
    
    def set_note_at_path(self, path, note_data):
        """Set note data at the given path."""
        current = self.note_structure
        for i, part in enumerate(path[:-1]):
            if part not in current:
                current[part] = {"children": {}}
            current = current[part].get("children", {})
        
        current[path[-1]] = note_data
    
    def select_note_by_name(self, name, parent_id=""):
        """Select a note in the tree by its name and parent."""
        for item_id in self.notes_tree.get_children(parent_id):
            if self.notes_tree.item(item_id)["text"] == name:
                self.notes_tree.selection_set(item_id)
                self.notes_tree.focus(item_id)
                self.notes_tree.see(item_id)
                self.on_note_select(None)
                return True
        return False
    
    def add_subnote(self):
        """Add a subnote to the selected note."""
        if not self.notes_tree.selection():
            messagebox.showerror("Error", "Please select a note first!")
            return
        self.add_note()
    
    def rename_note(self):
        """Rename the selected note."""
        if not self.notes_tree.selection():
            messagebox.showerror("Error", "Please select a note first!")
            return
        
        node_id = self.notes_tree.selection()[0]
        old_name = self.notes_tree.item(node_id)["text"]
        path = self.get_note_path(node_id)
        
        new_name = simpledialog.askstring("Rename Note", "Enter new name:", initialvalue=old_name)
        if not new_name or new_name == old_name:
            return
        
        # Save current note content first
        self.save_current_note()
        
        # Get the parent container
        if len(path) > 1:
            parent_path = path[:-1]
            parent = self.get_note_by_path(parent_path)
            children = parent.get("children", {})
            
            # Check if name exists
            if new_name in children:
                messagebox.showerror("Error", "A note with this name already exists here!")
                return
            
            # Rename the note in structure
            parent["children"][new_name] = parent["children"].pop(old_name)
        else:
            # Root note
            if new_name in self.note_structure:
                messagebox.showerror("Error", "A note with this name already exists!")
                return
            
            # Rename root note
            self.note_structure[new_name] = self.note_structure.pop(old_name)
        
        # Rename the content file
        old_file = self.get_note_filename(path)
        new_path = path[:-1] + [new_name]
        new_file = self.get_note_filename(new_path)
        
        if os.path.exists(old_file):
            # Read content from old file
            with open(old_file, "r", encoding="utf-8") as f:
                content = f.read()
            
            # Write content to new file
            with open(new_file, "w", encoding="utf-8") as f:
                f.write(content)
            
            # Delete old file
            os.remove(old_file)
        
        # Update current path if we're renaming the currently edited note
        if self.currently_editing_path and self.currently_editing_path[-1] == old_name:
            self.currently_editing_path = self.currently_editing_path[:-1] + [new_name]
            self.note_title.config(text=" → ".join(self.currently_editing_path))
        
        # Update structure and UI
        self.save_note_structure()
        self.populate_tree()
        
        # Re-select the renamed note
        parent_id = self.notes_tree.parent(node_id) if self.notes_tree.parent(node_id) else ""
        self.select_note_by_name(new_name, parent_id)
    
    def delete_note(self):
        """Delete the selected note."""
        if not self.notes_tree.selection():
            messagebox.showerror("Error", "Please select a note first!")
            return
        
        node_id = self.notes_tree.selection()[0]
        note_name = self.notes_tree.item(node_id)["text"]
        path = self.get_note_path(node_id)
        
        confirm = messagebox.askyesno("Confirm Delete", 
                                     f"Delete '{note_name}' and all its subnotes?")
        if not confirm:
            return
        
        # Delete note files recursively
        self.delete_note_files_recursive(path)
        
        # Remove from structure
        if len(path) > 1:
            parent_path = path[:-1]
            parent = self.get_note_by_path(parent_path)
            if "children" in parent and note_name in parent["children"]:
                del parent["children"][note_name]
        else:
            if note_name in self.note_structure:
                del self.note_structure[note_name]
        
        # Clear editor if deleting current note
        if self.currently_editing_path and self.currently_editing_path[-1] == note_name:
            self.text_editor.delete(1.0, tk.END)
            self.note_title.config(text="")
            self.currently_editing_path = None
        
        # Update structure and UI
        self.save_note_structure()
        self.populate_tree()
        self.status_var.set(f"Deleted note: {note_name}")
    
    def delete_note_files_recursive(self, path):
        """Delete a note and all its subnotes' files."""
        # Delete the note's own file
        note_file = self.get_note_filename(path)
        if os.path.exists(note_file):
            os.remove(note_file)
        
        # Find all possible child notes by examining the structure
        note = self.get_note_by_path(path)
        if isinstance(note, dict) and "children" in note:
            for child_name in note["children"]:
                child_path = path + [child_name]
                self.delete_note_files_recursive(child_path)
    
    def mark_dirty(self, event=None):
        """Mark note as changed and update status."""
        if self.currently_editing_path:
            self.is_dirty = True
            self.status_var.set("Edited - will autosave soon")
    
    def search_notes(self):
        """Search for notes by name or content."""
        search_term = simpledialog.askstring("Search Notes", "Enter search term:")
        if not search_term:
            return
        
        # Save current note first to ensure it's included in search
        self.save_current_note()
        
        results = []
        self.search_notes_recursive(self.note_structure, [], search_term.lower(), results)
        
        if not results:
            messagebox.showinfo("Search Results", "No matches found.")
            return
        
        # Show results in a new window
        result_window = tk.Toplevel(self.root)
        result_window.title(f"Search Results for '{search_term}'")
        result_window.geometry("600x400")
        
        # Create frame with scrollbar
        frame = tk.Frame(result_window)
        frame.pack(fill=tk.BOTH, expand=True)
        
        tree = ttk.Treeview(frame)
        tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        
        scrollbar = ttk.Scrollbar(frame, orient="vertical", command=tree.yview)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        tree.configure(yscrollcommand=scrollbar.set)
        
        tree["columns"] = ("match",)
        tree.column("#0", width=200)
        tree.column("match", width=380)
        tree.heading("#0", text="Note Path")
        tree.heading("match", text="Content Preview")
        
        # Add search results
        for i, (path, kind, match) in enumerate(results):
            path_str = " → ".join(path)
            
            # Prepare preview text
            if kind == "name":
                preview = f"[Matched title]: {match}"
            else:
                # Get context around the match
                lower_content = match.lower()
                search_pos = lower_content.find(search_term.lower())
                start = max(0, search_pos - 40)
                end = min(len(match), search_pos + len(search_term) + 40)
                
                # Get context with ellipses if needed
                preview = match[start:end]
                if start > 0:
                    preview = "..." + preview
                if end < len(match):
                    preview = preview + "..."
            
            tree.insert("", "end", text=path_str, values=(preview,))
        
        tree.bind("<Double-1>", lambda e: self.open_search_result(tree, result_window))
    
    def open_search_result(self, tree, window):
        """Open a note from search results."""
        selected = tree.selection()
        if not selected:
            return
        
        # Get the path from the text
        path_str = tree.item(selected[0], "text")
        path = path_str.split(" → ")
        
        # Find and select the note in the main tree
        self.select_note_by_path(path)
        window.destroy()
    
    def select_note_by_path(self, path):
        """Select a note in the tree by its path."""
        def find_node(parent, remaining_path):
            if not remaining_path:
                return parent
            
            for child in self.notes_tree.get_children(parent):
                if self.notes_tree.item(child)["text"] == remaining_path[0]:
                    self.notes_tree.item(child, open=True)
                    return find_node(child, remaining_path[1:])
            return None
        
        if not path:
            return
            
        root_node = find_node("", path)
        if root_node:
            self.notes_tree.selection_set(root_node)
            self.notes_tree.focus(root_node)
            self.notes_tree.see(root_node)
            self.on_note_select(None)
    
    def search_notes_recursive(self, nodes, current_path, term, results):
        """Recursively search notes."""
        for name, data in nodes.items():
            path = current_path + [name]
            
            # Check if name matches
            if term in name.lower():
                results.append((path, "name", name))
            
            # Check if content matches
            content = self.load_note_content(path)
            if term in content.lower():
                results.append((path, "content", content))
            
            # Search children
            if isinstance(data, dict) and "children" in data:
                self.search_notes_recursive(data["children"], path, term, results)
    
    def toggle_dark_mode(self):
        """Toggle between light and dark themes."""
        self.current_mode = DARK_MODE if self.current_mode == LIGHT_MODE else LIGHT_MODE
        self.apply_theme()
        self.save_theme()
    
    def apply_theme(self):
        """Apply the current theme to all widgets."""
        # Main window and panels
        self.root.config(bg=self.current_mode['bg'])
        self.main_pane.config(bg=self.current_mode['bg'])
        self.left_panel.config(bg=self.current_mode['bg'])
        self.right_panel.config(bg=self.current_mode['bg'])
        self.control_frame.config(bg=self.current_mode['bg'])
        self.button_frame.config(bg=self.current_mode['bg'])
        
        # Buttons
        for btn in [self.add_note_btn, self.add_subnote_btn, self.rename_btn, 
                   self.delete_btn, self.toggle_mode_btn, self.search_btn]:
            btn.config(
                bg=self.current_mode['button_bg'],
                fg=self.current_mode['button_fg']
            )
        
        # Text widgets
        self.text_editor.config(
            bg=self.current_mode['bg'],
            fg=self.current_mode['fg'],
            insertbackground=self.current_mode['fg']
        )
        self.note_title.config(
            bg=self.current_mode['bg'],
            fg=self.current_mode['fg']
        )
        
        # Status bar
        self.status_bar.config(
            bg=self.current_mode['bg'],
            fg=self.current_mode['fg']
        )
        
        # Treeview style
        style = ttk.Style()
        style.theme_use('default')
        style.configure("Treeview",
            background=self.current_mode['tree_bg'],
            foreground=self.current_mode['tree_fg'],
            fieldbackground=self.current_mode['tree_bg']
        )
        style.map("Treeview",
            background=[('selected', self.current_mode['tree_sel_bg'])],
            foreground=[('selected', self.current_mode['tree_sel_fg'])]
        )
    
    def save_theme(self):
        """Save the current theme to config."""
        config = {"theme": "dark" if self.current_mode == DARK_MODE else "light"}
        with open(CONFIG_FILE, "w") as f:
            json.dump(config, f)
    
    def load_theme(self):
        """Load theme from config."""
        if os.path.exists(CONFIG_FILE):
            with open(CONFIG_FILE, "r") as f:
                config = json.load(f)
                return DARK_MODE if config.get("theme") == "dark" else LIGHT_MODE
        return LIGHT_MODE
    
    def autosave(self):
        """Periodically save notes."""
        if self.is_dirty:
            self.save_current_note()
        self.root.after(self.autosave_interval, self.autosave)

if __name__ == "__main__":
    root = tk.Tk()
    check_for_update()
    app = NoteTakingApp(root)
    root.mainloop()