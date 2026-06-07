# SETs-Project
from pathlib import Path
from PIL import Image
import random
import math
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
folder = Path(r"C:\Users\bwijc\Studie\Wiskunde\Programmeren voor Wiskunde\kaarten")
images = {}
for file in folder.glob("*.gif"):
    images[file.stem] = Image.open(file)
print("Loaded", len(images), "images")


hand = SET.display_random_cards(images)
sets = SET.set_in_c_combination(hand)
print("Set found:", sets)
