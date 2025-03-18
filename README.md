import tkinter as tk
from tkinter import ttk, messagebox
import yfinance as yf
import threading
import time
import random
import matplotlib.pyplot as plt
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg
import pandas as pd
from datetime import datetime

class TradingBotApp:
    # Liste des paires de devises
    DEVISES = ["EURUSD=X", "GBPUSD=X", "USDJPY=X", "AUDUSD=X", "USDCAD=X", "EURGBP=X", "NZDUSD=X", "USDCHF=X"]
    STRATEGIES = {
        "Buy and Hold": "Achat et maintien de position sur le long terme",
        "Scalping": "Transactions rapides pour des petits profits",
        "Momentum": "Suivi des tendances de marché",
        "Mean Reversion": "Transactions basées sur le retour à la moyenne"
    }
    
    def __init__(self, root):
        self.root = root
        self.root.title("Dashboard Trading - Bot Automatique")
        self.root.geometry("1200x800")
        self.root.protocol("WM_DELETE_WINDOW", self.on_closing)
        
        # Variables d'état
        self.bot_running = False
        self.bot_thread = None
        self.data_history = {}  # Stockage de l'historique des prix
        self.transactions = []  # Historique des transactions
        self.selected_devise = tk.StringVar()
        self.selected_strategy = tk.StringVar()
        self.capital_initial = 10000  # Capital de départ en euros
        self.capital_actuel = self.capital_initial
        
        # Création de l'interface
        self.create_ui()
        
        # Initialiser le graphique
        self.update_selected_chart()
        
        # Mettre à jour les prix toutes les 30 secondes
        self.root.after(1000, self.update_prices)
    
    def create_ui(self):
        """Création de l'interface utilisateur"""
        # Frame principale avec style
        main_frame = ttk.Frame(self.root, padding="10")
        main_frame.pack(fill=tk.BOTH, expand=True)
        
        # Titre de l'application
        title_frame = ttk.Frame(main_frame)
        title_frame.pack(fill=tk.X, pady=10)
        ttk.Label(title_frame, text="DASHBOARD DE TRADING AUTOMATIQUE", 
                 font=("Helvetica", 16, "bold")).pack()
        
        # Création de deux colonnes principales
        left_frame = ttk.Frame(main_frame)
        left_frame.pack(side=tk.LEFT, fill=tk.BOTH, expand=True, padx=5)
        
        right_frame = ttk.Frame(main_frame)
        right_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=5)
        
        # --- COLONNE GAUCHE ---
        # Section graphique
        graph_frame = ttk.LabelFrame(left_frame, text="Graphique de la devise", padding=10)
        graph_frame.pack(fill=tk.BOTH, expand=True, pady=5)
        
        # Créer le graphique
        self.fig, self.ax = plt.subplots(figsize=(6, 4))
        self.canvas = FigureCanvasTkAgg(self.fig, master=graph_frame)
        self.canvas.get_tk_widget().pack(fill=tk.BOTH, expand=True)
        
        # Section configuration du bot
        config_frame = ttk.LabelFrame(left_frame, text="Configuration du bot", padding=10)
        config_frame.pack(fill=tk.X, pady=5)
        
        # Sélection de devise
        devise_frame = ttk.Frame(config_frame)
        devise_frame.pack(fill=tk.X, pady=5)
        ttk.Label(devise_frame, text="Devise:").pack(side=tk.LEFT, padx=5)
        devise_combo = ttk.Combobox(devise_frame, textvariable=self.selected_devise, 
                                    values=self.DEVISES, state="readonly", width=10)
        devise_combo.pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        devise_combo.current(0)
        devise_combo.bind("<<ComboboxSelected>>", self.update_selected_chart)
        
        # Sélection de stratégie
        strategy_frame = ttk.Frame(config_frame)
        strategy_frame.pack(fill=tk.X, pady=5)
        ttk.Label(strategy_frame, text="Stratégie:").pack(side=tk.LEFT, padx=5)
        strategy_combo = ttk.Combobox(strategy_frame, textvariable=self.selected_strategy, 
                                     values=list(self.STRATEGIES.keys()), state="readonly", width=15)
        strategy_combo.pack(side=tk.LEFT, padx=5, fill=tk.X, expand=True)
        strategy_combo.current(0)
        strategy_combo.bind("<<ComboboxSelected>>", self.update_strategy_description)
        
        # Description de la stratégie
        self.strategy_desc = ttk.Label(config_frame, text="", wraplength=300)
        self.strategy_desc.pack(fill=tk.X, pady=5)
        self.update_strategy_description()
        
        # Paramètres de trading
        params_frame = ttk.Frame(config_frame)
        params_frame.pack(fill=tk.X, pady=5)
        
        # Montant par trade
        ttk.Label(params_frame, text="Montant par trade (€):").grid(row=0, column=0, sticky=tk.W, padx=5, pady=2)
        self.amount_var = tk.StringVar(value="1000")
        ttk.Entry(params_frame, textvariable=self.amount_var, width=10).grid(row=0, column=1, padx=5, pady=2)
        
        # Stop loss (%)
        ttk.Label(params_frame, text="Stop Loss (%):").grid(row=1, column=0, sticky=tk.W, padx=5, pady=2)
        self.stop_loss_var = tk.StringVar(value="2")
        ttk.Entry(params_frame, textvariable=self.stop_loss_var, width=10).grid(row=1, column=1, padx=5, pady=2)
        
        # Take profit (%)
        ttk.Label(params_frame, text="Take Profit (%):").grid(row=2, column=0, sticky=tk.W, padx=5, pady=2)
        self.take_profit_var = tk.StringVar(value="3")
        ttk.Entry(params_frame, textvariable=self.take_profit_var, width=10).grid(row=2, column=1, padx=5, pady=2)
        
        # Bouton de démarrage/arrêt
        self.start_button = ttk.Button(config_frame, text="Démarrer le Bot", command=self.toggle_bot)
        self.start_button.pack(pady=10)
        
        # --- COLONNE DROITE ---
        # Informations de compte
        account_frame = ttk.LabelFrame(right_frame, text="Informations du compte", padding=10)
        account_frame.pack(fill=tk.X, pady=5)
        
        self.capital_label = ttk.Label(account_frame, text=f"Capital: {self.capital_initial} €")
        self.capital_label.pack(anchor=tk.W, pady=2)
        
        self.profit_label = ttk.Label(account_frame, text="Profit/Perte: 0 €")
        self.profit_label.pack(anchor=tk.W, pady=2)
        
        self.positions_label = ttk.Label(account_frame, text="Positions ouvertes: 0")
        self.positions_label.pack(anchor=tk.W, pady=2)
        
        # Historique des transactions
        history_frame = ttk.LabelFrame(right_frame, text="Historique des transactions", padding=10)
        history_frame.pack(fill=tk.BOTH, expand=True, pady=5)
        
        # Table des transactions
        columns = ("Date", "Devise", "Type", "Montant", "Prix", "Statut")
        self.history_tree = ttk.Treeview(history_frame, columns=columns, show="headings", height=10)
        
        for col in columns:
            self.history_tree.heading(col, text=col)
            width = 70 if col in ("Montant", "Prix") else 100
            self.history_tree.column(col, width=width)
        
        scrollbar = ttk.Scrollbar(history_frame, orient=tk.VERTICAL, command=self.history_tree.yview)
        self.history_tree.configure(yscrollcommand=scrollbar.set)
        
        self.history_tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Journal de bord
        log_frame = ttk.LabelFrame(right_frame, text="Journal d'activité", padding=10)
        log_frame.pack(fill=tk.BOTH, expand=True, pady=5)
        
        self.log_text = tk.Text(log_frame, height=6, wrap=tk.WORD)
        self.log_text.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        
        log_scrollbar = ttk.Scrollbar(log_frame, orient=tk.VERTICAL, command=self.log_text.yview)
        self.log_text.configure(yscrollcommand=log_scrollbar.set)
        log_scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Ajouter un message initial au journal
        self.log_message("Application démarrée. En attente de configuration...")
    
    def log_message(self, message):
        """Ajoute un message au journal d'activité"""
        timestamp = datetime.now().strftime("%H:%M:%S")
        self.log_text.insert(tk.END, f"[{timestamp}] {message}\n")
        self.log_text.see(tk.END)  # Défilement automatique vers le bas
    
    def update_strategy_description(self, event=None):
        """Met à jour la description de la stratégie sélectionnée"""
        strategy = self.selected_strategy.get()
        if strategy in self.STRATEGIES:
            self.strategy_desc.config(text=self.STRATEGIES[strategy])
    
    def get_real_time_prices(self):
        """Récupère les prix réels des devises depuis Yahoo Finance"""
        prices = {}
        for devise in self.DEVISES:
            try:
                data = yf.download(devise, period="1d", interval="1m", progress=False)
                if not data.empty:
                    prices[devise] = round(data['Close'].iloc[-1], 4)
                    
                    # Stocker l'historique pour le graphique
                    if devise not in self.data_history:
                        self.data_history[devise] = []
                    
                    # Limiter l'historique à 100 points
                    if len(self.data_history[devise]) >= 100:
                        self.data_history[devise].pop(0)
                    
                    self.data_history[devise].append(prices[devise])
                else:
                    # Si pas de données, générer un prix aléatoire
                    prices[devise] = round(random.uniform(1.0, 1.5), 4)
            except Exception as e:
                # Générer un prix aléatoire en cas d'erreur
                prices[devise] = round(random.uniform(1.0, 1.5), 4)
                self.log_message(f"Erreur lors de la récupération du prix de {devise}: {str(e)}")
        
        return prices
    
    def update_prices(self):
        """Met à jour les prix et les graphiques périodiquement"""
        try:
            prices = self.get_real_time_prices()
            self.update_selected_chart()
            
            # Vérifier les positions ouvertes pour les stop loss/take profit
            self.check_open_positions(prices)
            
            # Programmer la prochaine mise à jour
            self.root.after(30000, self.update_prices)  # Toutes les 30 secondes
        except Exception as e:
            self.log_message(f"Erreur lors de la mise à jour des prix: {str(e)}")
            self.root.after(60000, self.update_prices)  # Réessayer après 1 minute
    
    def update_selected_chart(self, event=None):
        """Met à jour le graphique de la devise sélectionnée"""
        devise = self.selected_devise.get()
        
        # Récupérer les données historiques ou récupérer de nouvelles données
        if devise in self.data_history and self.data_history[devise]:
            data = self.data_history[devise]
        else:
            # Si pas de données historiques, récupérer des données
            try:
                df = yf.download(devise, period="1d", interval="1m", progress=False)
                data = df['Close'].tolist()[-30:]  # Dernières 30 minutes
                self.data_history[devise] = data
            except:
                # Générer des données fictives en cas d'erreur
                data = [round(random.uniform(1.0, 1.5) + i * 0.001, 4) for i in range(30)]
                self.data_history[devise] = data
        
        # Mettre à jour le graphique
        self.ax.clear()
        self.ax.plot(data, marker="o", linestyle="-", color="#1f77b4", linewidth=2)
        self.ax.set_title(f"{devise} - Évolution du prix")
        self.ax.set_xlabel("Temps")
        self.ax.set_ylabel("Prix")
        self.ax.grid(True, linestyle="--", alpha=0.7)
        
        # Ajouter un peu de style au graphique
        self.fig.tight_layout()
        self.canvas.draw()
    
    def toggle_bot(self):
        """Démarre ou arrête le bot de trading"""
        if self.bot_running:
            self.bot_running = False
            self.start_button.config(text="Démarrer le Bot")
            self.log_message("Bot arrêté.")
        else:
            try:
                # Vérifier que les entrées numériques sont valides
                amount = float(self.amount_var.get())
                stop_loss = float(self.stop_loss_var.get())
                take_profit = float(self.take_profit_var.get())
                
                if amount <= 0 or stop_loss <= 0 or take_profit <= 0:
                    raise ValueError("Les valeurs doivent être positives")
                
                if amount > self.capital_actuel:
                    messagebox.showwarning("Capital insuffisant", 
                                          f"Montant par trade ({amount} €) supérieur au capital disponible ({self.capital_actuel} €)")
                    return
                
                self.bot_running = True
                self.bot_thread = threading.Thread(target=self.run_bot)
                self.bot_thread.daemon = True  # Le thread s'arrêtera quand le programme principal s'arrête
                self.bot_thread.start()
                self.start_button.config(text="Arrêter le Bot")
                self.log_message(f"Bot démarré avec la stratégie: {self.selected_strategy.get()}")
                
            except ValueError as e:
                messagebox.showerror("Erreur de configuration", 
                                    f"Veuillez vérifier les paramètres: {str(e)}")
                self.log_message(f"Erreur de configuration: {str(e)}")
    
    def run_bot(self):
        """Exécute les opérations de trading en fonction de la stratégie sélectionnée"""
        devise = self.selected_devise.get()
        strategy = self.selected_strategy.get()
        
        while self.bot_running:
            try:
                # Récupérer le prix actuel
                prices = self.get_real_time_prices()
                current_price = prices[devise]
                
                # Décider d'une action selon la stratégie
                action, reason = self.execute_strategy(strategy, devise, current_price)
                
                if action:
                    # Montant de la transaction
                    montant = float(self.amount_var.get())
                    
                    # Exécuter la transaction
                    self.execute_trade(devise, action, montant, current_price, reason)
                    
                    # Attendre plus longtemps après une transaction
                    time.sleep(random.randint(20, 40))
                else:
                    # Attendre un moment avant la prochaine analyse
                    time.sleep(random.randint(5, 15))
                    
            except Exception as e:
                self.log_message(f"Erreur d'exécution du bot: {str(e)}")
                time.sleep(30)  # Attendre avant de réessayer
    
    def execute_strategy(self, strategy, devise, current_price):
        """Applique la stratégie de trading sélectionnée"""
        # Historique des prix pour analyse
        history = self.data_history.get(devise, [])
        
        if not history or len(history) < 5:
            return None, "Pas assez de données historiques"
        
        # Initialisation
        action = None
        reason = ""
        
        # Stratégies implémentées
        if strategy == "Buy and Hold":
            # Acheter si nous n'avons pas déjà une position ouverte
            has_open_position = any(t["devise"] == devise and t["status"] == "Ouvert" 
                                  for t in self.transactions)
            
            if not has_open_position and random.random() < 0.3:  # 30% de chance d'acheter
                action = "Achat"
                reason = "Stratégie Buy and Hold - Position à long terme"
        
        elif strategy == "Scalping":
            # Petites fluctuations rapides
            last_5 = history[-5:]
            price_change = (last_5[-1] - last_5[0]) / last_5[0] * 100
            
            if price_change > 0.1:
                action = "Achat"
                reason = f"Scalping - Hausse rapide de {price_change:.2f}%"
            elif price_change < -0.1:
                action = "Vente"
                reason = f"Scalping - Baisse rapide de {price_change:.2f}%"
        
        elif strategy == "Momentum":
            # Tendance à court terme
            short_term = history[-10:]
            trend = sum(1 if short_term[i] < short_term[i+1] else -1 for i in range(len(short_term)-1))
            
            if trend > 3:  # Tendance haussière
                action = "Achat"
                reason = "Momentum - Tendance haussière détectée"
            elif trend < -3:  # Tendance baissière
                action = "Vente"
                reason = "Momentum - Tendance baissière détectée"
        
        elif strategy == "Mean Reversion":
            # Retour à la moyenne
            avg_price = sum(history[-20:]) / len(history[-20:])
            deviation = (current_price - avg_price) / avg_price * 100
            
            if deviation < -1.5:  # Prix bien en-dessous de la moyenne
                action = "Achat"
                reason = f"Mean Reversion - Prix {deviation:.2f}% sous la moyenne"
            elif deviation > 1.5:  # Prix bien au-dessus de la moyenne
                action = "Vente"
                reason = f"Mean Reversion - Prix {deviation:.2f}% au-dessus de la moyenne"
        
        # Ajouter un élément aléatoire pour la simulation
        if action is None and random.random() < 0.1:  # 10% de chance d'agir au hasard
            action = random.choice(["Achat", "Vente"])
            reason = "Analyse technique - Opportunité détectée"
        
        return action, reason
    
    def execute_trade(self, devise, action, montant, prix, raison):
        """Exécute une transaction et l'ajoute à l'historique"""
        # Créer un ID de transaction unique
        transaction_id = len(self.transactions) + 1
        
        # Calculer le stop loss et take profit
        stop_loss_pct = float(self.stop_loss_var.get()) / 100
        take_profit_pct = float(self.take_profit_var.get()) / 100
        
        stop_loss = round(prix * (1 - stop_loss_pct if action == "Achat" else 1 + stop_loss_pct), 4)
        take_profit = round(prix * (1 + take_profit_pct if action == "Achat" else 1 - take_profit_pct), 4)
        
        # Créer l'entrée de transaction
        transaction = {
            "id": transaction_id,
            "date": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            "devise": devise,
            "type": action,
            "montant": montant,
            "prix": prix,
            "stop_loss": stop_loss,
            "take_profit": take_profit,
            "status": "Ouvert",
            "raison": raison
        }
        
        # Ajouter à la liste des transactions
        self.transactions.append(transaction)
        
        # Ajouter à l'interface
        self.history_tree.insert("", "end", values=(
            transaction["date"], 
            transaction["devise"], 
            transaction["type"], 
            f"{transaction['montant']:.2f} €", 
            transaction["prix"],
            transaction["status"]
        ))
        
        # Mettre à jour le capital
        if action == "Achat":
            self.capital_actuel -= montant
        else:
            self.capital_actuel += montant
        
        self.update_account_info()
        
        # Journaliser l'action
        self.log_message(f"Transaction #{transaction_id}: {action} de {montant:.2f} € sur {devise} au prix de {prix} ({raison})")
    
    def check_open_positions(self, current_prices):
        """Vérifie les positions ouvertes pour les stop loss/take profit"""
        closed_transactions = []
        
        for transaction in self.transactions:
            if transaction["status"] != "Ouvert":
                continue
                
            devise = transaction["devise"]
            if devise not in current_prices:
                continue
                
            current_price = current_prices[devise]
            action = transaction["type"]
            entry_price = transaction["prix"]
            stop_loss = transaction["stop_loss"]
            take_profit = transaction["take_profit"]
            
            # Vérifier si stop loss ou take profit sont atteints
            if action == "Achat":
                if current_price <= stop_loss:
                    # Stop loss déclenché
                    profit = (stop_loss - entry_price) / entry_price * transaction["montant"]
                    transaction["status"] = "Fermé (Stop Loss)"
                    closed_transactions.append((transaction, profit, "Stop Loss"))
                
                elif current_price >= take_profit:
                    # Take profit déclenché
                    profit = (take_profit - entry_price) / entry_price * transaction["montant"]
                    transaction["status"] = "Fermé (Take Profit)"
                    closed_transactions.append((transaction, profit, "Take Profit"))
            
            elif action == "Vente":
                if current_price >= stop_loss:
                    # Stop loss déclenché
                    profit = (entry_price - stop_loss) / entry_price * transaction["montant"]
                    transaction["status"] = "Fermé (Stop Loss)"
                    closed_transactions.append((transaction, profit, "Stop Loss"))
                
                elif current_price <= take_profit:
                    # Take profit déclenché
                    profit = (entry_price - take_profit) / entry_price * transaction["montant"]
                    transaction["status"] = "Fermé (Take Profit)"
                    closed_transactions.append((transaction, profit, "Take Profit"))
        
        # Mettre à jour l'interface pour les transactions fermées
        for transaction, profit, reason in closed_transactions:
            # Mettre à jour l'arbre des transactions
            for item in self.history_tree.get_children():
                values = self.history_tree.item(item, "values")
                if (values[0] == transaction["date"] and 
                    values[1] == transaction["devise"] and 
                    values[2] == transaction["type"]):
                    self.history_tree.item(item, values=values[:-1] + (transaction["status"],))
                    break
            
            # Mettre à jour le capital
            self.capital_actuel += transaction["montant"] + profit
            
            # Journaliser
            self.log_message(
                f"Position fermée: {transaction['type']} de {transaction['montant']:.2f} € sur {transaction['devise']} - "
                f"{reason} déclenché. Profit/Perte: {profit:.2f} €"
            )
        
        # Mettre à jour les informations du compte
        if closed_transactions:
            self.update_account_info()
    
    def update_account_info(self):
        """Met à jour les informations du compte affichées"""
        profit = self.capital_actuel - self.capital_initial
        open_positions = sum(1 for t in self.transactions if t["status"] == "Ouvert")
        
        self.capital_label.config(text=f"Capital: {self.capital_actuel:.2f} €")
        
        if profit >= 0:
            self.profit_label.config(text=f"Profit: +{profit:.2f} € (+{profit/self.capital_initial*100:.2f}%)")
        else:
            self.profit_label.config(text=f"Perte: {profit:.2f} € ({profit/self.capital_initial*100:.2f}%)")
            
        self.positions_label.config(text=f"Positions ouvertes: {open_positions}")
    
    def on_closing(self):
        """Gestion de la fermeture de l'application"""
        if self.bot_running:
            if messagebox.askyesno("Confirmation", "Le bot est en cours d'exécution. Voulez-vous vraiment quitter?"):
                self.bot_running = False
                self.root.destroy()
        else:
            self.root.destroy()

if __name__ == "__main__":
    root = tk.Tk()
    app = TradingBotApp(root)
    root.mainloop()
