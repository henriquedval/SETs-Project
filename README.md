from pathlib import Path
from PIL import Image, ImageTk
import random
import tkinter as tk
import matplotlib.pyplot as plt
from itertools import combinations

# -----------------------------
# LOAD IMAGES --> make sure to change the folder directory!
# -----------------------------
folder = Path(r"C:\Users\bwijc\Studie\Wiskunde\Programmeren voor Wiskunde\kaarten")
images = {}
for file in folder.glob("*.gif"):
    images[file.stem] = Image.open(file)
print("Loaded", len(images), "images")

class SET:
    # -----------------------------
    # CLASS DICTIONARIES (so we know which variable is coupled to which integer), INITIALISING
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
    # SET ALGORITHMS: CHECKING IF 3 CARDS FORM A SET, FINDING ALL SETS IN A HAND 
    # -----------------------------
    @staticmethod
    def set_or_not(c1, c2, c3):             
        for i in range(len(c1)): 
            if not ((c1[i] != c2[i] and c1[i] != c3[i] and c2[i] !=c3[i]) or (c1[i] == c2[i] == c3[i])) : 
                return False 
        else: return True

    @staticmethod
    def set_in_c_combination(hand_names):
        all_sets = []
        for combination in combinations(hand_names, 3):
            c1 = SET.from_string(combination[0]).vector
            c2 = SET.from_string(combination[1]).vector
            c3 = SET.from_string(combination[2]).vector
            if SET.set_or_not(c1, c2, c3):
                all_sets.append(combination)
        return all_sets if all_sets else None
    
    @staticmethod
    def complete_set(c1,c2):    # this function can find the card that would complete a SET
        result = []
        for i in range(len(c1)):
            if c1[i] == c2[i]:
                result.append(c1[i])
            if c1[i] != c2[i]:
                result.append(({0,1,2}-{c1[i],c2[i]}).pop())
        return tuple(result)


# -----------------------------
# INTERACTIVE GAME (TKINTER)
# -----------------------------
class SetGame(tk.Tk):
    def __init__(self, images, n=12):
        super().__init__()
        self.title("SET")
        self.images = images
        self.card_vectors = {name: SET.from_string(name).vector for name in self.images}
        self.n = n
        self.photos = {}          # keep PhotoImage refs alive (avoids blank cards)
        self.buttons = {}         # name -> Button
        self.selected = set()     # currently selected card names (max 3)
        self.default_bg = None    # filled in when the first button is built
        self.player_score = 0     # sets found by the player
        self.computer_score = 0   # sets found by the computer
        self.waiting_for_replace = False   # making sure that the selected cards cannot be changed during the time they are replaced
        self.cards_to_replace = []

        # create a shuffled deck and deal 12 cards from the deck
        # first hand should contain at least one SET
        while True:
            self.deck = list(self.images.keys())
            random.shuffle(self.deck)
            self.hand = [self.deck.pop() for _ in range(12)]
            if self.find_all_sets(self.hand):
                break
        # keeping track of the positions of the cards, so that when new cards will be drawn, the positions in the plot wont change    
        self.card_positions = {}
        for i in range(len(self.hand)):
            name = self.hand[i]
            self.card_positions[name] = i

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
        for card in list(self.selected):
            self.style(card, False)
        self.selected.clear()
        active = list(self.buttons.keys())
        sets = self.find_all_sets(active)
        if sets:
            self.computer_score += 1
            self.update_score()
            self.waiting_for_replace = True
            self.cards_to_replace = list(sets[0])   # the set the computer "found"
            for card in self.cards_to_replace:
                self.buttons[card].config(bg="red", relief=tk.SUNKEN)
            self.status.config(text="Time's up! The computer found a set.", fg="red")
        else:
            self.status.config(text="Time's up! No set was there, cards refreshed.", fg="red")
            self.cards_to_replace = active[:3]         # no set on the board -> refresh 3 cards
        self.after(2000, self.replace_set)     # waits 2 seconds to display the found SET, then replaces cards.     

    def on_click(self, name):
        if self.waiting_for_replace:    # if cards are being swapped, no new ones can be selected
            return
        # a verdict is already on screen (3 picked) -> this click starts a new try
        if len(self.selected) == 3:
            self.clear_selection()

        if name in self.selected:
            self.selected.remove(name)
            self.style(name, False)
        else:
            self.selected.add(name)
            self.style(name, True)
            if len(self.selected) == 3:
                self.check_set()

    def check_set(self):
        c1, c2, c3 = (self.card_vectors[name] for name in self.selected)
        if SET.set_or_not(c1, c2, c3):
            self.player_score += 1
            self.update_score()
            self.status.config(text="This is a set. Good job!", fg="green")
            self.waiting_for_replace = True
            self.cards_to_replace = self.selected.copy()
            self.after(2000, self.replace_set) 
        else:
            self.status.config(text="This is not a set...", fg="red")
    
    def find_all_sets(self, hand_names):    
        vectors = [self.card_vectors[name] for name in hand_names]
        vector_to_name = {self.card_vectors[name]: name for name in hand_names}
        all_sets = []
        for combination in combinations(vectors,2):
            needed = SET.complete_set(*combination)
            if needed in vector_to_name:
                current_set = frozenset({vector_to_name[combination[0]],vector_to_name[combination[1]],vector_to_name[needed]})     #because it finds the same set 3 times, this guarantees only one is added to all_sets
                all_sets.append(current_set)
        return all_sets if all_sets else None
    
    #def find_one_set(self, hand_names):        # this is essentialy the same as the function find_all_sets and is therefore not needed
    #    vectors = [self.card_vectors[name] for name in hand_names]
    #    vector_to_name = {self.card_vectors[name]: name for name in hand_names}
    #    for combination in combinations(vectors, 2):
    #        if (needed := SET.complete_set(*combination)) in vector_to_name:
    #            current_set = (vector_to_name[combination[0]],vector_to_name[combination[1]],vector_to_name[needed])
    #    return current_set
    
    def clear_selection(self):
        for name in self.selected:
            self.style(name, False)
        self.selected.clear()
        self.status.config(text="Pick 3 cards", fg="black")

    def end_of_game(self):
        self.game_over = True
        if self.timer_job is not None:
            self.after_cancel(self.timer_job)
            self.timer_job = None
        for btn in self.buttons.values():
            btn.config(state=tk.DISABLED)
        if self.player_score > self.computer_score:
            self.status.config(text="No more SETs available. You Win!", fg="green")
        elif self.player_score == self.computer_score:
            self.status.config(text="No more SETs available. Draw!", fg="yellow")
        else:
            self.status.config(text="No more SETs available. Computer wins!", fg="red")

    def replace_set(self):
        for old_card in self.cards_to_replace:
            if len(self.deck) == 0:                 # if deck is empty, the game continues with 3 less cards in the hand
                self.buttons[old_card].destroy()
                del self.buttons[old_card]
                del self.card_positions[old_card]
                continue
            new_card = self.deck.pop()
            self.hand[self.card_positions[old_card]] = new_card    # selected SET from hand is changed to the top cards of the deck
            
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
        self.selected = set()
        self.cards_to_replace = []
        self.waiting_for_replace = False

        if len(self.deck) == 0 and not self.find_all_sets(active_cards):
            self.end_of_game()
        else:
            self.status.config(text="Pick 3 cards", fg="black")    
            if not self.game_over:
                self.start_timer(self.time_limit)

SetGame(images).mainloop()
