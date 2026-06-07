# SETs-Project
from pathlib import Path
from PIL import Image, ImageTk
import random
import math
import tkinter as tk
import matplotlib.pyplot as plt


class SET:
    # -----------------------------
    # CLASS DICTIONARIES
    # -----------------------------
    colors = {"green": 0, "purple": 1, "red": 2}
    numbers = {"1": 0, "2": 1, "3": 2}
    shadings = {"empty": 0, "filled": 1, "shaded": 2}
    symbols = {"diamond": 0, "oval": 1, "squiggle": 2}

    # -----------------------------
    # INIT
    # -----------------------------
    def __init__(self, color, number, shading, symbol):
        self.vector = (color, number, shading, symbol)

    # -----------------------------
    # FROM STRINGS TO CARDS
    # -----------------------------
    @classmethod
    def from_string(cls, card):
        # color
        if "green" in card:
            color = 0
        elif "purple" in card:
            color = 1
        elif "red" in card:
            color = 2

        # number
        if "1" in card:
            number = 0
        elif "2" in card:
            number = 1
        elif "3" in card:
            number = 2

        # shading
        if "empty" in card:
            shading = 0
        elif "filled" in card:
            shading = 1
        elif "shaded" in card:
            shading = 2

        # symbol
        if "diamond" in card:
            symbol = 0
        elif "oval" in card:
            symbol = 1
        elif "squiggle" in card:
            symbol = 2

        return cls(color, number, shading, symbol)
    
    # -----------------------------
    # COUPLING NAMES TO CORRESPONDING VECTORS AND VICE VERSA
    # -----------------------------
    @staticmethod
    def names_to_vectors(hand_names):
        mapping = {}
        for name in hand_names:
            card = SET.from_string(name)
            mapping[name] = card.vector
        return mapping

    @staticmethod
    def vectors_to_names(hand_names):
        mapping = {}
        for name in hand_names:
            card = SET.from_string(name)
            mapping[card.vector] = name
        return mapping
    
    # -----------------------------
    # SET ALGORITHMS
    # -----------------------------
    
    @staticmethod
    def set_or_not(c1, c2, c3):             
        for i in range(len(c1)): 
            if not ((c1[i] != c2[i] and c1[i] != c3[i] and c2[i] !=c3[i]) or (c1[i] == c2[i] == c3[i])) : return False 
        else: return True

    @staticmethod
    def generate_all_combinations(vectors):
        import itertools
        return list(itertools.combinations(vectors, 3))

    @staticmethod
    def set_in_c_combination(hand_names):
        cards = [SET.from_string(name) for name in hand_names]
        vectors = [card.vector for card in cards]
        vector_to_name = {card.vector: name for card, name in zip(cards, hand_names)}
        all_combinations = SET.generate_all_combinations(vectors)
        all_sets = []
        for combination in all_combinations:                       
            if SET.set_or_not(*combination):
                current_set = tuple(vector_to_name[v] for v in combination)
                all_sets.append(tuple(current_set))

        return all_sets if all_sets else None
    
    # -----------------------------
    # DISPLAY RANDOM CARDS
    # -----------------------------
    def display_random_cards(images, n=12):
        chosen = random.sample(list(images.keys()), n)
        rows = 3
        cols = 4
        fig, axes = plt.subplots(rows, cols, figsize=(cols * 2, rows * 3))
        axes = axes.flatten()
        for i in range(n):
            name = chosen[i]
            axes[i].imshow(images[name])
            axes[i].set_title(name, fontsize=8)
            axes[i].axis("off")
        for j in range(n, len(axes)):
            axes[j].axis("off")
        plt.tight_layout()
        plt.show()
        return chosen

# -----------------------------
# LOAD IMAGES
# -----------------------------
folder = Path(r"C:\Users\henri\OneDrive\Desktop\UU Lock In Period\Python Course\kaarten\kaarten")
images = {}
for file in folder.glob("*.gif"):
    images[file.stem] = Image.open(file)
print("Loaded", len(images), "images")


# -----------------------------
# INTERACTIVE GAME (TKINTER)
# -----------------------------
class SetGame(tk.Tk):
    def __init__(self, images, n=12):
        super().__init__()
        self.title("SET")
        self.images = images
        self.n = n
        self.photos = {}          # keep PhotoImage refs alive (avoids blank cards)
        self.buttons = {}         # name -> Button
        self.selected = []        # currently selected card names (max 3)
        self.default_bg = None    # filled in when the first button is built

        # deal a hand that is guaranteed to contain at least one set
        self.hand = self.draw_hand()
        self.vectors = SET.names_to_vectors(self.hand)   # name -> vector

        self.build_grid()
        self.status = tk.Label(self, text="Pick 3 cards", font=("Arial", 14))
        self.status.grid(row=3, column=0, columnspan=4, pady=10)

    def draw_hand(self):
        names = list(self.images.keys())
        while True:
            hand = random.sample(names, self.n)
            if SET.set_in_c_combination(hand):   # at least one set exists -> solvable
                return hand

    def build_grid(self):
        for i, name in enumerate(self.hand):
            photo = ImageTk.PhotoImage(self.images[name])
            self.photos[name] = photo
            btn = tk.Button(self, image=photo, relief=tk.RAISED, bd=3,
                            command=lambda n=name: self.on_click(n))
            btn.grid(row=i // 4, column=i % 4, padx=4, pady=4)
            self.buttons[name] = btn
            if self.default_bg is None:
                self.default_bg = btn.cget("background")

    def style(self, name, selected):
        if selected:
            self.buttons[name].config(relief=tk.SUNKEN, bg="gold")
        else:
            self.buttons[name].config(relief=tk.RAISED, bg=self.default_bg)

    def on_click(self, name):
        # a verdict is already on screen (3 picked) -> this click starts a new try
        if len(self.selected) == 3:
            self.clear_selection()

        if name in self.selected:
            self.selected.remove(name)
            self.style(name, False)
        else:
            self.selected.append(name)
            self.style(name, True)
            if len(self.selected) == 3:
                self.check_set()

    def check_set(self):
        c1, c2, c3 = (self.vectors[name] for name in self.selected)
        if SET.set_or_not(c1, c2, c3):
            self.status.config(text="This is a set. Good job!", fg="green")
        else:
            self.status.config(text="This is not a set...", fg="red")

    def clear_selection(self):
        for name in self.selected:
            self.style(name, False)
        self.selected = []
        self.status.config(text="Pick 3 cards", fg="black")


SetGame(images).mainloop()
