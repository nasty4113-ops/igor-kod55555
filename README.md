# igor-kod55555
github_user_finder/
├── .gitignore
├── README.md
├── requirements.txt
├── main.py
├── favorites.json
└── github_api.py
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.env
venv/
env/
.venv/

# IDE
.vscode/
.idea/
*.swp
*.swo

# Данные
favorites.json
*.log

# OS
.DS_Store
Thumbs.db
requests==2.31.0
Pillow==10.1.0
import requests
import json
import os

class GitHubAPI:
    """Класс для работы с GitHub API"""
    
    BASE_URL = "https://api.github.com"
    
    def __init__(self):
        self.favorites_file = "favorites.json"
        self.favorites = self.load_favorites()
    
    def search_users(self, username, per_page=30):
        """Поиск пользователей GitHub по имени"""
        url = f"{self.BASE_URL}/search/users"
        params = {
            'q': username,
            'per_page': per_page,
            'sort': 'followers'
        }
        
        try:
            response = requests.get(url, params=params, timeout=10)
            response.raise_for_status()
            data = response.json()
            
            users = []
            for item in data.get('items', []):
                user_details = self.get_user_details(item['login'])
                if user_details:
                    users.append(user_details)
            
            return {
                'success': True,
                'total_count': data.get('total_count', 0),
                'users': users
            }
            
        except requests.exceptions.RequestException as e:
            return {
                'success': False,
                'error': f"Ошибка при поиске: {str(e)}",
                'users': []
            }
    
    def get_user_details(self, username):
        """Получение детальной информации о пользователе"""
        url = f"{self.BASE_URL}/users/{username}"
        
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            data = response.json()
            
            return {
                'login': data.get('login'),
                'name': data.get('name', 'Не указано'),
                'avatar_url': data.get('avatar_url'),
                'html_url': data.get('html_url'),
                'public_repos': data.get('public_repos', 0),
                'followers': data.get('followers', 0),
                'following': data.get('following', 0),
                'bio': data.get('bio', 'Нет описания'),
                'company': data.get('company', 'Не указано'),
                'location': data.get('location', 'Не указано'),
                'created_at': data.get('created_at', 'Неизвестно')
            }
            
        except requests.exceptions.RequestException:
            return None
    
    def load_favorites(self):
        """Загрузка избранных пользователей из JSON файла"""
        if os.path.exists(self.favorites_file):
            try:
                with open(self.favorites_file, 'r', encoding='utf-8') as file:
                    return json.load(file)
            except json.JSONDecodeError:
                return []
        return []
    
    def save_favorites(self):
        """Сохранение избранных пользователей в JSON файл"""
        with open(self.favorites_file, 'w', encoding='utf-8') as file:
            json.dump(self.favorites, file, indent=4, ensure_ascii=False)
    
    def add_to_favorites(self, user):
        """Добавление пользователя в избранное"""
        if not any(fav['login'] == user['login'] for fav in self.favorites):
            self.favorites.append(user)
            self.save_favorites()
            return True
        return False
    
    def remove_from_favorites(self, username):
        """Удаление пользователя из избранного"""
        self.favorites = [fav for fav in self.favorites if fav['login'] != username]
        self.save_favorites()
    
    def is_favorite(self, username):
        """Проверка, находится ли пользователь в избранном"""
        return any(fav['login'] == username for fav in self.favorites)
    
    def get_favorites(self):
        """Получение списка избранных пользователей"""
        return self.favorites
        import tkinter as tk
from tkinter import ttk, messagebox, scrolledtext
import threading
from datetime import datetime
import webbrowser
from github_api import GitHubAPI

class GitHubUserFinder:
    """Главный класс GUI-приложения"""
    
    def __init__(self, root):
        self.root = root
        self.root.title("GitHub User Finder")
        self.root.geometry("1200x700")
        self.root.configure(bg='#f0f0f0')
        
        self.api = GitHubAPI()
        self.current_users = []
        
        self.setup_ui()
        self.load_favorites_view()
    
    def setup_ui(self):
        """Настройка пользовательского интерфейса"""
        
        # Главный контейнер
        main_container = ttk.PanedWindow(self.root, orient=tk.VERTICAL)
        main_container.pack(fill=tk.BOTH, expand=True, padx=10, pady=10)
        
        # Верхняя панель поиска
        search_frame = ttk.LabelFrame(main_container, text="Поиск пользователей GitHub", padding=10)
        main_container.add(search_frame, weight=0)
        
        # Поле ввода и кнопка поиска
        search_controls = ttk.Frame(search_frame)
        search_controls.pack(fill=tk.X, expand=True)
        
        ttk.Label(search_controls, text="Введите имя пользователя:").pack(side=tk.LEFT, padx=(0, 10))
        
        self.search_entry = ttk.Entry(search_controls, width=40, font=('Arial', 12))
        self.search_entry.pack(side=tk.LEFT, padx=(0, 10))
        self.search_entry.bind('<Return>', lambda e: self.search_users())
        
        self.search_button = ttk.Button(search_controls, text="Поиск", command=self.search_users)
        self.search_button.pack(side=tk.LEFT, padx=(0, 10))
        
        self.clear_button = ttk.Button(search_controls, text="Очистить", command=self.clear_search)
        self.clear_button.pack(side=tk.LEFT)
        
        # Статус поиска
        self.status_label = ttk.Label(search_frame, text="Готов к поиску", foreground='gray')
        self.status_label.pack(anchor=tk.W, pady=(5, 0))
        
        # Средняя панель с результатами
        results_frame = ttk.LabelFrame(main_container, text="Результаты поиска", padding=10)
        main_container.add(results_frame, weight=1)
        
        # Создаем Treeview для отображения результатов
        columns = ('username', 'name', 'repos', 'followers', 'following', 'action')
        self.results_tree = ttk.Treeview(results_frame, columns=columns, show='headings', selectmode='browse')
        
        # Определяем заголовки столбцов
        self.results_tree.heading('username', text='Username')
        self.results_tree.heading('name', text='Имя')
        self.results_tree.heading('repos', text='Репозитории')
        self.results_tree.heading('followers', text='Подписчики')
        self.results_tree.heading('following', text='Подписки')
        self.results_tree.heading('action', text='Действие')
        
        # Настройка ширины столбцов
        self.results_tree.column('username', width=150)
        self.results_tree.column('name', width=200)
        self.results_tree.column('repos', width=100)
        self.results_tree.column('followers', width=100)
        self.results_tree.column('following', width=100)
        self.results_tree.column('action', width=100)
        
        # Добавляем scrollbar
        scrollbar = ttk.Scrollbar(results_frame, orient=tk.VERTICAL, command=self.results_tree.yview)
        self.results_tree.configure(yscrollcommand=scrollbar.set)
        
        self.results_tree.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Привязываем двойной клик для открытия профиля
        self.results_tree.bind('<Double-Button-1>', self.open_user_profile)
        
        # Правая панель с избранным и деталями
        right_panel = ttk.Frame(results_frame)
        right_panel.pack(side=tk.RIGHT, fill=tk.Y, padx=(10, 0))
        
        # Панель избранного
        favorites_frame = ttk.LabelFrame(right_panel, text="Избранные пользователи", padding=10)
        favorites_frame.pack(fill=tk.BOTH, expand=True)
        
        self.favorites_listbox = tk.Listbox(favorites_frame, height=20, width=30, font=('Arial', 10))
        self.favorites_listbox.pack(side=tk.LEFT, fill=tk.BOTH, expand=True)
        
        fav_scrollbar = ttk.Scrollbar(favorites_frame, orient=tk.VERTICAL, command=self.favorites_listbox.yview)
        self.favorites_listbox.configure(yscrollcommand=fav_scrollbar.set)
        fav_scrollbar.pack(side=tk.RIGHT, fill=tk.Y)
        
        # Привязываем клик для показа деталей избранного пользователя
        self.favorites_listbox.bind('<<ListboxSelect>>', self.show_favorite_details)
        
        # Кнопки управления избранным
        fav_buttons = ttk.Frame(right_panel)
        fav_buttons.pack(fill=tk.X, pady=(10, 0))
        
        self.remove_fav_btn = ttk.Button(fav_buttons, text="Удалить из избранного", 
                                        command=self.remove_from_favorites, state=tk.DISABLED)
        self.remove_fav_btn.pack(side=tk.LEFT, expand=True)
        
        # Нижняя панель деталей
        details_frame = ttk.LabelFrame(main_container, text="Детальная информация", padding=10)
        main_container.add(details_frame, weight=0)
        
        self.details_text = scrolledtext.ScrolledText(details_frame, height=8, wrap=tk.WORD, 
                                                      font=('Arial', 10))
        self.details_text.pack(fill=tk.BOTH, expand=True)
    
    def search_users(self):
        """Поиск пользователей GitHub"""
        username = self.search_entry.get().strip()
        
        # Проверка на пустое поле
        if not username:
            messagebox.showwarning("Предупреждение", "Поле поиска не должно быть пустым!")
            return
        
        # Отключаем кнопку на время поиска
        self.search_button.configure(state=tk.DISABLED)
        self.status_label.configure(text=f"Поиск пользователей: {username}...", foreground='blue')
        
        # Запускаем поиск в отдельном потоке
        thread = threading.Thread(target=self.perform_search, args=(username,))
        thread.daemon = True
        thread.start()
    
    def perform_search(self, username):
        """Выполнение поиска в отдельном потоке"""
        result = self.api.search_users(username)
        
        # Обновляем UI в главном потоке
        self.root.after(0, self.update_search_results, result, username)
    
    def update_search_results(self, result, username):
        """Обновление результатов поиска в UI"""
        # Очищаем старые результаты
        for item in self.results_tree.get_children():
            self.results_tree.delete(item)
        
        if result['success']:
            self.current_users = result['users']
            
            if self.current_users:
                for user in self.current_users:
                    is_fav = self.api.is_favorite(user['login'])
                    action_text = "★ В избранном" if is_fav else "☆ Добавить"
                    
                    values = (
                        user['login'],
                        user['name'] or 'Не указано',
                        user['public_repos'],
                        user['followers'],
                        user['following'],
                        action_text
                    )
                    
                    item = self.results_tree.insert('', tk.END, values=values)
                    
                    # Подсвечиваем избранных пользователей
                    if is_fav:
                        self.results_tree.item(item, tags=('favorite',))
                
                self.results_tree.tag_configure('favorite', background='#fff3cd')
                self.status_label.configure(
                    text=f"Найдено пользователей: {len(self.current_users)} из {result['total_count']}", 
                    foreground='green'
                )
            else:
                self.status_label.configure(text="Пользователи не найдены", foreground='orange')
        else:
            messagebox.showerror("Ошибка", result['error'])
            self.status_label.configure(text="Ошибка поиска", foreground='red')
        
        self.search_button.configure(state=tk.NORMAL)
        
        # Привязываем обработчик одинарного клика
        self.results_tree.bind('<ButtonRelease-1>', self.handle_result_click)
    
    def handle_result_click(self, event):
        """Обработчик клика по результату"""
        selection = self.results_tree.selection()
        if not selection:
            return
        
        item = selection[0]
        values = self.results_tree.item(item)['values']
        
        if not values:
            return
        
        # Определяем, в какой столбец кликнули
        column = self.results_tree.identify_column(event.x)
        
        if column == '#6':  # Столбец действий
            username = values[0]
            self.toggle_favorite(username)
        else:
            # Показываем детали пользователя
            self.show_user_details(username)
    
    def toggle_favorite(self, username):
        """Добавление/удаление из избранного"""
        if self.api.is_favorite(username):
            self.api.remove_from_favorites(username)
            messagebox.showinfo("Информация", f"Пользователь {username} удален из избранного")
        else:
            # Находим пользователя в текущих результатах
            user_to_add = None
            for user in self.current_users:
                if user['login'] == username:
                    user_to_add = user
                    break
            
            if user_to_add:
                if self.api.add_to_favorites(user_to_add):
                    messagebox.showinfo("Информация", f"Пользователь {username} добавлен в избранное")
        
        # Обновляем отображение
        self.load_favorites_view()
        self.refresh_current_view()
    
    def show_user_details(self, username):
        """Показать детальную информацию о пользователе"""
        user = None
        
        # Ищем пользователя в текущих результатах или избранном
        for u in self.current_users:
            if u['login'] == username:
                user = u
                break
        
        if not user:
            for u in self.api.get_favorites():
                if u['login'] == username:
                    user = u
                    break
        
        if user:
            self.display_user_details(user)
    
    def display_user_details(self, user):
        """Отображение детальной информации о пользователе"""
        self.details_text.delete(1.0, tk.END)
        
        details = f"""
=== Информация о пользователе GitHub ===

👤 Логин: {user['login']}
📛 Имя: {user['name']}
📝 Биография: {user['bio']}
🏢 Компания: {user['company']}
📍 Местоположение: {user['location']}

📊 Статистика:
   • Публичные репозитории: {user['public_repos']}
   • Подписчики: {user['followers']}
   • Подписки: {user['following']}

📅 Дата регистрации: {user['created_at'][:10] if user['created_at'] else 'Неизвестно'}
🔗 Профиль: {user['html_url']}
"""
        
        self.details_text.insert(1.0, details)
    
    def open_user_profile(self, event):
        """Открыть профиль пользователя в браузере"""
        selection = self.results_tree.selection()
        if selection:
            item = selection[0]
            values = self.results_tree.item(item)['values']
            if values:
                username = values[0]
                url = f"https://github.com/{username}"
                webbrowser.open(url)
    
    def load_favorites_view(self):
        """Загрузка избранных пользователей в список"""
        self.favorites_listbox.delete(0, tk.END)
        favorites = self.api.get_favorites()
        
        for user in favorites:
            display_text = f"{user['login']} - {user['name'] or 'Без имени'}"
            self.favorites_listbox.insert(tk.END, display_text)
    
    def show_favorite_details(self, event):
        """Показать детали выбранного избранного пользователя"""
        selection = self.favorites_listbox.curselection()
        if selection:
            self.remove_fav_btn.configure(state=tk.NORMAL)
            index = selection[0]
            favorites = self.api.get_favorites()
            if 0 <= index < len(favorites):
                self.display_user_details(favorites[index])
        else:
            self.remove_fav_btn.configure(state=tk.DISABLED)
    
    def remove_from_favorites(self):
        """Удаление пользователя из избранного"""
        selection = self.favorites_listbox.curselection()
        if selection:
            index = selection[0]
            favorites = self.api.get_favorites()
            if 0 <= index < len(favorites):
                username = favorites[index]['login']
                self.api.remove_from_favorites(username)
                self.load_favorites_view()
                self.refresh_current_view()
                self.details_text.delete(1.0, tk.END)
                self.remove_fav_btn.configure(state=tk.DISABLED)
                messagebox.showinfo("Информация", f"Пользователь {username} удален из избранного")
    
    def refresh_current_view(self):
        """Обновление текущего отображения результатов"""
        if self.current_users:
            username = self.search_entry.get().strip()
            if username:
                self.update_search_results(
                    {'success': True, 'total_count': len(self.current_users), 'users': self.current_users},
                    username
                )
    
    def clear_search(self):
        """Очистка поля поиска и результатов"""
        self.search_entry.delete(0, tk.END)
        for item in self.results_tree.get_children():
            self.results_tree.delete(item)
        self.current_users = []
        self.details_text.delete(1.0, tk.END)
        self.status_label.configure(text="Готов к поиску", foreground='gray')

def main():
    """Точка входа в приложение"""
    root = tk.Tk()
    
    # Настройка стилей
    style = ttk.Style()
    style.theme_use('clam')
    
    app = GitHubUserFinder(root)
    root.mainloop()

if __name__ == "__main__":
    main()
