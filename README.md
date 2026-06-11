from pathlib import Path
from PIL import Image, ImageTk
import random
import math
import tkinter as tk
import matplotlib.pyplot as plt


class SET:
    # -----------------------------
    # CLASS DICTIONARIES, INITIALISING, FUNCTIONS TO CONVERT CLASS OBJECT TO VECTOR AND VICE VERSA
    # -----------------------------
    colors = {"green": 0, "purple": 1, "red": 2}
    numbers = {"1": 0, "2": 1, "3": 2}
    shadings = {"empty": 0, "filled": 1, "shaded": 2}
    symbols = {"diamond": 0, "oval": 1, "squiggle": 2}

    def __init__(self, color, number, shading, symbol):
        self.vector = (color, number, shading, symbol)

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
    # SET ALGORITHMS: CHECKING IF 3 CARDS FORM A SET, FINDING ALL SETS IN A HAND 
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
        vector_to_name = SET.vectors_to_names(hand_names)
        all_combinations = SET.generate_all_combinations(vectors)
        all_sets = []
        for combination in all_combinations:                       
            if SET.set_or_not(*combination):
                current_set = tuple(vector_to_name[v] for v in combination)
                all_sets.append(tuple(current_set))

        return all_sets if all_sets else None
    
    # -----------------------------
    # DISPLAY RANDOM CARDS --> not used anymore, just stored 
    # -----------------------------
    #def display_random_cards(images, n=12):
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
        self.player_score = 0     # sets found by the player
        self.computer_score = 0   # sets found by the computer

        # create a shuffled deck and deal 12 cards from the deck
        # first hand should contain at least one SET
        while True:
            self.deck = list(self.images.keys())
            random.shuffle(self.deck)
            self.hand = [self.deck.pop() for _ in range(12)]
            if SET.set_in_c_combination(self.hand):
                break
        # keeping track of the positions of the cards, so that when new cards will be drawn, the positions in the plot wont change    
        self.card_positions = {}
        for i in range(len(self.hand)):
            name = self.hand[i]
            self.card_positions[name] = i

        self.vectors = SET.names_to_vectors(self.hand)

        # header on top: score and timer (kept here so they stay visible)
        self.header = tk.Frame(self)
        self.header.grid(row=0, column=0, columnspan=4, pady=8)
        self.score_label = tk.Label(self.header, text="Player: 0    Computer: 0", font=("Arial", 14))
        self.score_label.grid(row=0, column=0, padx=20)
        self.timer_label = tk.Label(self.header, text="Time: --", font=("Arial", 14))
        self.timer_label.grid(row=0, column=1, padx=20)

        self.build_grid()

        self.status = tk.Label(self, text="Select 3 cards that make a SET", font=("Arial", 14))
        self.status.grid(row=4, column=0, columnspan=4, pady=10)

        self.time_limit = None    # seconds per round, set when difficulty is chosen
        self.remaining = 0        # seconds left in the current round
        self.timer_job = None     # id of the scheduled countdown callback
        self.round_marker = 0     # player_score at the start of the current round
        self.game_over = False    # set to True when the game ends

        self.difficulties = {"Easy": 60, "Medium": 30, "Hard": 15}
        seconds = self.choose_difficulty()
        if seconds is None:                    # panel closed without choosing
            seconds = self.difficulties["Medium"]
        self.start_timer(seconds)

    def build_grid(self):
        for i, name in enumerate(self.hand):
            photo = ImageTk.PhotoImage(self.images[name])
            self.photos[name] = photo
            btn = tk.Button(self, image=photo, relief=tk.RAISED, bd=3,
                            command=lambda n=name: self.on_click(n))
            btn.grid(row=i // 4 + 1, column=i % 4, padx=4, pady=4)
            self.buttons[name] = btn
            if self.default_bg is None:
                self.default_bg = btn.cget("background")

    def style(self, name, selected):
        if selected:
            self.buttons[name].config(relief=tk.SUNKEN, bg="gold")
        else:
            self.buttons[name].config(relief=tk.RAISED, bg=self.default_bg)

    def update_score(self):
        self.score_label.config(text=f"Player: {self.player_score}    Computer: {self.computer_score}")

    def choose_difficulty(self):
        panel = tk.Toplevel(self)
        panel.title("Select difficulty")
        panel.grab_set()                       # block the board until a choice is made
        tk.Label(panel, text="Choose a difficulty", font=("Arial", 14)).pack(padx=20, pady=10)
        row = tk.Frame(panel)
        row.pack(padx=20, pady=(0, 15))
        chosen = {"seconds": None}
        def pick(seconds):
            chosen["seconds"] = seconds
            panel.destroy()
        for i, (label, seconds) in enumerate(self.difficulties.items()):
            tk.Button(row, text=label, width=8,
                      command=lambda s=seconds: pick(s)).grid(row=0, column=i, padx=5)
        self.wait_window(panel)                # wait here until a button is clicked
        return chosen["seconds"]

    def start_timer(self, seconds):
        if self.timer_job is not None:
            self.after_cancel(self.timer_job)
        self.time_limit = seconds
        self.remaining = seconds
        self.round_marker = self.player_score
        self.timer_label.config(text=f"Time: {self.remaining}")
        self.timer_job = self.after(1000, self.update_timer)

    def update_timer(self):
        if self.player_score != self.round_marker:   # player found a set -> fresh round, reset clock
            self.round_marker = self.player_score
            self.remaining = self.time_limit
            self.timer_label.config(text=f"Time: {self.remaining}")
            self.timer_job = self.after(1000, self.update_timer)
            return
        self.remaining -= 1
        self.timer_label.config(text=f"Time: {self.remaining}")
        if self.remaining <= 0:
            self.time_up()
        else:
            self.timer_job = self.after(1000, self.update_timer)

    def time_up(self):
        if self.game_over:
            return
        self.timer_job = None
        active = list(self.buttons.keys())
        sets = SET.set_in_c_combination(active)
        if sets:
            self.computer_score += 1
            self.update_score()
            self.selected = list(sets[0])      # the set the computer "found"
            found = True
        else:
            self.selected = active[:3]         # no set on the board -> refresh 3 cards
            found = False
        self.replace_set()
        if not self.game_over:
            if found:
                self.status.config(text="Time's up! The computer found a set.", fg="red")
            else:
                self.status.config(text="Time's up! No set was there, cards refreshed.", fg="red")
            self.start_timer(self.time_limit)

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
            self.player_score += 1
            self.update_score()
            self.status.config(text="This is a set. Good job!", fg="green")
            self.after(2000, self.replace_set)  # after 2 seconds, the set is replaced --> need to add function to keep track of the scores
        else:
            self.status.config(text="This is not a set...", fg="red")

    def clear_selection(self):
        for name in self.selected:
            self.style(name, False)
        self.selected = []
        self.status.config(text="Pick 3 cards", fg="black")

    def end_of_game(self):
        self.game_over = True
        if self.timer_job is not None:
            self.after_cancel(self.timer_job)
            self.timer_job = None
        for btn in self.buttons.values():
            btn.config(state=tk.DISABLED)
        if self.player_score > self.computer_score:
            self.status.config(text="You Win!", fg="green")
        else:
            self.status.config(text="Loser!", fg="red")

    def replace_set(self):
        for old_card in self.selected:
            if len(self.deck) == 0:                                # if deck is empty, the game continues with 3 less cards in the hand
                self.buttons[old_card].destroy()
                del self.buttons[old_card]
                del self.card_positions[old_card]
                continue
            new_card = self.deck.pop()
            self.hand[self.card_positions[old_card]] = new_card    # selected SET from is changed to the last cards of the deck
            
            self.card_positions[new_card] = self.card_positions[old_card]   # keep track of the position of the new cards
            del self.card_positions[old_card]

            photo = ImageTk.PhotoImage(self.images[new_card])
            self.photos[new_card] = photo

            # update existing button --> should look like it is not selected
            btn = self.buttons[old_card]
            btn.config(image=photo, relief=tk.RAISED, bg=self.default_bg, command=lambda n=new_card: self.on_click(n))
            self.buttons[new_card] = btn
            del self.buttons[old_card]

        active_cards = list(self.buttons.keys())
        self.vectors = SET.names_to_vectors(active_cards)
        self.selected = []

        if len(self.deck) == 0 and not SET.set_in_c_combination(self.hand):
            self.end_of_game()
        else:
            self.status.config(text="Pick 3 cards", fg="black")    

SetGame(images).mainloop()
