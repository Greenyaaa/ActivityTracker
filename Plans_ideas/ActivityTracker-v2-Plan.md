# ActivityTracker v2 — План разработки
**Stack:** Vanilla JS + Supabase + GitHub Pages  
**Supabase проект:** новый (отдельный от проекта #1)

---

## 0. Обзор архитектуры

```
GitHub Pages (index.html)
  ↕ Supabase JS SDK (браузер)
Supabase Project #2
  ├── Auth         — Supabase Email Auth (фейковый email = ник + домен)
  ├── PostgreSQL   — все данные (профили, тренировки, стрики, друзья, группы)
  ├── RLS Policies — каждый видит только то, что разрешено
  └── Storage      — (опционально) кастомные аватарки
```

**Принцип аутентификации — Ник + PIN:**
Supabase Auth используется со схемой `nickname@activitytracker.app` как email и PIN как пароль. Пользователь никогда не видит «email» — только ник и PIN. Сессия хранится в `localStorage` автоматически через Supabase SDK. При входе с нового устройства — вводит тот же ник + PIN, попадает в тот же аккаунт.

```
Регистрация:  sb.auth.signUp({ email: `${ник}@activitytracker.app`, password: PIN })
Вход:         sb.auth.signInWithPassword({ email: `${ник}@activitytracker.app`, password: PIN })
Сессия:       хранится в localStorage автоматически (Supabase SDK)
Новое устройство: вход по нику + PIN → та же сессия, те же данные
```

> **Ограничение:** Supabase по умолчанию отправляет email-подтверждение. Нужно **отключить** `Email Confirm` в Supabase Dashboard → Auth → Settings → `Enable email confirmations = OFF`.

---

## 1. База данных — схема таблиц

### 1.1 `profiles`
| Поле | Тип | Описание |
|---|---|---|
| `id` | uuid PK | = `auth.uid()` |
| `nickname` | text unique | выбирается при онбординге |
| `avatar_seed` | text | seed для генерации аватарки (DiceBear) |
| `avatar_animal` | text | тип зверушки (cat / dog / fox / bear / rabbit / ...) |
| `role` | enum | `user` / `trainer` / `admin` |
| `is_active` | bool | false у тренера до подтверждения Админа |
| `trainer_status` | enum | `pending` / `approved` / `rejected` / null |
| `default_goal` | int | цель по умолчанию (7000/8000/9000/10000) |
| `created_at` | timestamptz | |

### 1.2 `workouts`
| Поле | Тип | Описание |
|---|---|---|
| `id` | uuid PK | |
| `user_id` | uuid FK → profiles | |
| `date` | date | день тренировки |
| `exercises` | jsonb | `{walk: 30, ell: 20, bike: 0, ...}` |
| `total_points` | int | итог сессии |
| `goal` | int | цель на этот день (7000/8000/...) |
| `goal_achieved` | bool | `total_points >= goal` |
| `created_at` | timestamptz | |

> Уникальный constraint: `(user_id, date)` — одна запись в день (upsert при повторном сохранении)

### 1.3 `streaks`
| Поле | Тип | Описание |
|---|---|---|
| `user_id` | uuid PK FK → profiles | |
| `current_streak` | int | текущая серия дней |
| `longest_streak` | int | рекорд всех времён |
| `last_goal_date` | date | дата последнего выполнения цели |
| `updated_at` | timestamptz | |

### 1.4 `friendships`
| Поле | Тип | Описание |
|---|---|---|
| `id` | uuid PK | |
| `requester_id` | uuid FK → profiles | кто отправил |
| `addressee_id` | uuid FK → profiles | кому отправил |
| `status` | enum | `pending` / `accepted` / `declined` |
| `created_at` | timestamptz | |

> Уникальный constraint: `(requester_id, addressee_id)`

### 1.5 `groups`
| Поле | Тип | Описание |
|---|---|---|
| `id` | uuid PK | |
| `trainer_id` | uuid FK → profiles | только тренер может создать |
| `name` | text | название группы |
| `description` | text | |
| `created_at` | timestamptz | |

### 1.6 `group_members`
| Поле | Тип | Описание |
|---|---|---|
| `id` | uuid PK | |
| `group_id` | uuid FK → groups | |
| `user_id` | uuid FK → profiles | |
| `status` | enum | `invited` / `accepted` / `declined` |
| `joined_at` | timestamptz | |

### 1.7 `trainer_approvals` (лог)
| Поле | Тип | Описание |
|---|---|---|
| `id` | uuid PK | |
| `trainer_id` | uuid FK → profiles | |
| `reviewed_by` | uuid FK → profiles | Admin UUID |
| `status` | enum | `pending` / `approved` / `rejected` |
| `admin_note` | text | комментарий Админа |
| `created_at` | timestamptz | |

---

## 2. RLS политики (Row Level Security)

```sql
-- profiles: читать можно публично (для поиска по нику),
-- писать только себе
CREATE POLICY "profiles_select" ON profiles FOR SELECT USING (true);
CREATE POLICY "profiles_insert" ON profiles FOR INSERT WITH CHECK (auth.uid() = id);
CREATE POLICY "profiles_update" ON profiles FOR UPDATE USING (auth.uid() = id);

-- workouts: видит только сам пользователь + его друзья + его тренер
CREATE POLICY "workouts_own" ON workouts FOR ALL USING (auth.uid() = user_id);
CREATE POLICY "workouts_friends" ON workouts FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM friendships
    WHERE status = 'accepted'
    AND ((requester_id = auth.uid() AND addressee_id = workouts.user_id)
      OR (addressee_id = auth.uid() AND requester_id = workouts.user_id))
  )
);
CREATE POLICY "workouts_trainer" ON workouts FOR SELECT USING (
  EXISTS (
    SELECT 1 FROM group_members gm
    JOIN groups g ON g.id = gm.group_id
    WHERE gm.user_id = workouts.user_id
    AND gm.status = 'accepted'
    AND g.trainer_id = auth.uid()
  )
);

-- friendships: видят оба участника
CREATE POLICY "friendships_select" ON friendships FOR SELECT
  USING (requester_id = auth.uid() OR addressee_id = auth.uid());
CREATE POLICY "friendships_insert" ON friendships FOR INSERT
  WITH CHECK (requester_id = auth.uid());
CREATE POLICY "friendships_update" ON friendships FOR UPDATE
  USING (addressee_id = auth.uid());

-- groups: тренер создаёт и управляет своими группами
CREATE POLICY "groups_trainer_all" ON groups FOR ALL USING (trainer_id = auth.uid());
CREATE POLICY "groups_member_select" ON groups FOR SELECT USING (
  EXISTS (SELECT 1 FROM group_members WHERE group_id = id AND user_id = auth.uid())
);

-- group_members: тренер управляет, участник видит своё
CREATE POLICY "group_members_trainer" ON group_members FOR ALL USING (
  EXISTS (SELECT 1 FROM groups WHERE id = group_id AND trainer_id = auth.uid())
);
CREATE POLICY "group_members_self" ON group_members FOR SELECT
  USING (user_id = auth.uid());
CREATE POLICY "group_members_accept" ON group_members FOR UPDATE
  USING (user_id = auth.uid());

-- trainer_approvals: тренер видит свою заявку, Админ видит все
CREATE POLICY "approvals_trainer" ON trainer_approvals FOR SELECT
  USING (trainer_id = auth.uid());
CREATE POLICY "approvals_admin" ON trainer_approvals FOR ALL USING (
  EXISTS (SELECT 1 FROM profiles WHERE id = auth.uid() AND role = 'admin')
);
```

---

## 3. Онбординг — первый запуск

### 3.1 Определение состояния при загрузке

```
Загрузка страницы
  ↓
sb.auth.getSession()
  ├── Нет сессии → показать экран [Вход / Регистрация]
  └── Есть сессия → загрузить профиль из profiles
        ├── Профиль не найден → показать онбординг (ник, аватарка, роль)
        └── Профиль найден
              ├── is_active = false (тренер на модерации) → экран ожидания
              └── is_active = true → главный экран трекера
```

### 3.2 Экран входа / регистрации

Два таба на одном экране:

**Таб «Новый аккаунт»**
1. Выбор зверушки и аватарки
   - 8–10 аватарок (DiceBear API: `api.dicebear.com/7.x/{style}/svg?seed={random}`)
   - Стили: `adventurer` / `lorelei` / `micah` / `notionists` / `pixel-art`
   - Кнопка «Перемешать» → новые случайные
2. Ввод ника — debounce-проверка уникальности в реальном времени
3. Придумать PIN (4–6 цифр) + повторить для подтверждения
4. Выбор роли (см. ниже)
5. Кнопка «Создать аккаунт» → `signUp`

**Таб «Войти»**
- Поле: Ник
- Поле: PIN
- Кнопка «Войти» → `signInWithPassword`
- Ошибка неверного ника/PIN → «Неверный ник или PIN»

### 3.3 Выбор роли при регистрации

| Роль | Кто видит | Поведение после регистрации |
|---|---|---|
| 👤 Пользователь | Все | `is_active = true` сразу, доступ к трекеру |
| 🏋️ Тренер | Все | `is_active = false`, экран ожидания Админа |
| 🛡️ Админ | Не отображается в UI | Назначается вручную через SQL |

### 3.4 Экран ожидания (тренер)
- Аватарка + ник пользователя
- Сообщение: _«Ваш аккаунт тренера ожидает подтверждения Администратора. Обычно это занимает до 24 часов.»_
- Supabase Realtime: `channel.on('postgres_changes', ...)` на строку профиля
- Как только Админ одобряет → `is_active = true` → автоматический переход на главный экран без перезагрузки

### 3.5 Настройка Supabase Auth (один раз)

```
Supabase Dashboard → Authentication → Settings:
  ✅ Enable Email provider = ON
  ❌ Enable email confirmations = OFF   ← важно!
  ❌ Enable phone confirmations = OFF
  Minimum password length = 4
```

### 3.6 Установка Админа (один раз после регистрации)

```sql
-- Выполнить в Supabase SQL Editor
-- Сначала зарегистрироваться через UI, затем найти свой UUID:
SELECT id FROM profiles WHERE nickname = 'твой_ник';

-- Назначить Админом:
UPDATE profiles
SET role = 'admin', is_active = true
WHERE nickname = 'твой_ник';
```

---

## 4. Goal Streak — система серий

### Логика подсчёта (выполняется при сохранении тренировки)

```javascript
async function updateStreak(userId, goalAchieved) {
  const today = new Date().toISOString().split('T')[0];
  const yesterday = new Date(Date.now() - 86400000).toISOString().split('T')[0];

  const { data: streak } = await sb.from('streaks')
    .select('*').eq('user_id', userId).single();

  if (!goalAchieved) return; // цель не выполнена — стрик не меняется

  let newCurrent = 1;
  if (streak?.last_goal_date === yesterday) {
    newCurrent = (streak.current_streak || 0) + 1; // продолжение серии
  } else if (streak?.last_goal_date === today) {
    return; // уже засчитано сегодня
  }
  // если last_goal_date < yesterday → серия прервана, newCurrent = 1

  const newLongest = Math.max(newCurrent, streak?.longest_streak || 0);

  await sb.from('streaks').upsert({
    user_id: userId,
    current_streak: newCurrent,
    longest_streak: newLongest,
    last_goal_date: today,
    updated_at: new Date().toISOString()
  });

  checkMilestone(newCurrent); // показать поздравление если нужно
}
```

### Milestone точки прогресса

| Стрик | Медаль | Сообщение |
|---|---|---|
| 1 день | 🌱 Росток | «Первый шаг сделан! Начало положено» |
| 3 дня | 🔥 Огонёк | «3 дня подряд — привычка начинает формироваться!» |
| 7 дней | ⭐ Неделя | «Целая неделя! Ты молодец, так держать» |
| 14 дней | 💪 Две недели | «Две недели — это уже характер» |
| 21 день | 🏅 Привычка | «21 день — учёные говорят, привычка сформирована!» |
| 28 дней | 🏆 Почти месяц | «28 дней без остановки — ты машина» |
| 29/30/31 | 👑 Месяц | «Полный месяц! Абсолютный рекорд» |

> 29/30/31 определяется через `new Date(year, month, 0).getDate()` — последний день текущего месяца.

**UI компонент стрика:**
- Горизонтальная шкала с отмеченными milestone точками (1, 3, 7, 14, 21, 28, 31)
- Текущая позиция (🔥) движется по шкале
- При достижении milestone — анимированный toast с медалью и сообщением
- Строка под шкалой: `🔥 7 дней подряд  |  Рекорд: 14 дней`

---

## 5. Социальные функции

### 5.1 Поиск пользователей
- Поиск по нику: `ILIKE '%query%'` по `profiles`
- Карточка результата: аватарка + ник + роль-иконка
- Кнопка «Добавить в друзья» / «Запрос отправлен» / «Уже друзья» / «Вы» (себя не показываем)
- Тренеры отмечены 🏋️, только активные (`is_active = true`)

### 5.2 Запросы в друзья

```
A отправляет → INSERT friendships {requester: A, addressee: B, status: 'pending'}
B получает badge-уведомление на вкладке «Друзья»
B принимает → UPDATE status = 'accepted'  → оба видят тренировки друг друга
B отклоняет → UPDATE status = 'declined'  → A может отправить снова позже
```

### 5.3 Лента друзей
- Карточка каждого друга: аватарка, ник, прогресс-бар сегодня, баллы, стрик
- Цель выполнена → зелёная рамка + ✅
- Ещё нет тренировки сегодня → серая карточка «Ещё не тренировался»
- Сортировка: выполнившие цель → тренирующиеся → без тренировки

---

## 6. Панель тренера

**Доступно только `role = 'trainer' AND is_active = true`**

| Функция | Описание |
|---|---|
| Создать группу | INSERT в `groups` |
| Пригласить пользователя | Поиск по нику → INSERT `group_members` (status='invited') |
| Дашборд группы | Прогресс всех участников за сегодня и 7 дней |
| Добавить в друзья | Стандартный флоу `friendships` |

**Дашборд группы:**
- Средний % выполнения цели по группе за 7 дней
- Кто не тренировался 3+ дня — выделено 🔴
- Кто на стрике 7+ дней — выделено 🔥

---

## 7. Панель Админа

**Доступно только `role = 'admin'`**

| Функция | Описание |
|---|---|
| Список заявок тренеров | `profiles WHERE trainer_status = 'pending'` |
| Подтвердить тренера | `UPDATE SET is_active=true, trainer_status='approved'` + INSERT в `trainer_approvals` |
| Отклонить | `UPDATE SET trainer_status='rejected'` + INSERT в `trainer_approvals` |
| Добавить заметку | Поле admin_note при принятии решения |

---

## 8. Структура файлов проекта

```
activity-tracker-v2/          ← новый GitHub репозиторий
├── index.html                ← единственный файл (всё внутри)
│     ├── <style>             — CSS (CSS variables, dark mode)
│     ├── <div#app>           — все экраны
│     └── <script>
│           ├── config        — Supabase URL + ANON_KEY
│           ├── auth          — вход/регистрация по нику+PIN, сессия
│           ├── onboarding    — выбор аватарки, ника, роли
│           ├── tracker       — слайдеры, баллы, сохранение тренировки
│           ├── streak        — подсчёт серий, milestone-уведомления
│           ├── social        — друзья, поиск, запросы, лента
│           ├── trainer       — группы, дашборд тренера
│           └── admin         — заявки тренеров
└── README.md
```

---

## 9. Экраны и навигация

```
[ Вход / Регистрация ]
  ├── Таб «Новый аккаунт» → аватарка → ник → PIN → роль → signUp
  └── Таб «Войти»         → ник → PIN → signInWithPassword
        ↓
[ Ожидание ] (только тренер до одобрения)
        ↓
[ Главная — Трекер ]
  ├── Стрик-виджет (🔥 N дней | Рекорд: N)
  ├── Слайдеры активности
  ├── Прогресс-бар + баллы
  └── Кнопка «Сохранить тренировку»

[ История ]
  └── Карточки по дням: дата / активности / баллы / цель ✅❌

[ Друзья ]
  ├── Поиск по нику
  ├── Входящие запросы 🔴
  ├── Мои друзья + прогресс сегодня
  └── Исходящие запросы

[ Группы ] (только тренер)
  ├── Мои группы
  └── Дашборд группы → прогресс участников

[ Админ ] (только Админ)
  └── Заявки тренеров → подтвердить / отклонить + заметка

[ Профиль ]
  ├── Аватарка + ник
  ├── Статистика (всего тренировок, лучший стрик, любимая активность)
  └── Кнопка «Выйти» → sb.auth.signOut()
```

---

## 10. Порядок реализации (фазы)

> **Легенда:** ✅ сделано · 🔧 в процессе · [ ] не начато

### Фаза 1 — Основа и Auth ✅ готово
- ✅ Создать Supabase проект #2
- ✅ Отключить email-подтверждение (Sign In / Providers → User Signups)
- ✅ Создать все таблицы + RLS политики
- ✅ Экран входа / регистрации (ник + PIN, два таба, 3 шага)
- ✅ Онбординг: аватарка (DiceBear), ник, PIN, роль
- ✅ Трекер (слайдеры, цели 7–10k, Advanced mode 3x)
- ✅ Сохранение тренировки upsert + подгрузка при открытии
- ✅ Экран «История»
- ✅ Supabase SDK подключён
- ✅ Назначить себя Админом через SQL

> **Заметки:** домен `.local` → `.app`; PIN паддится `@AT2`; добавлен `default_goal` в profiles.

### Фаза 2 — Стрики ✅ готово
- ✅ `updateStreak` при сохранении тренировки
- ✅ Streak-виджет с milestone-шкалой (1→3→7→14→21→28→31)
- ✅ Toast при достижении milestone
- ✅ Стрик в ленте друзей

### Фаза 3 — Социальные функции ✅ готово
- ✅ Поиск по нику (debounce, ILIKE)
- ✅ Запросы в друзья (отправить / принять / отклонить)
- ✅ Лента прогресса друзей с сортировкой
- ✅ Badge-счётчик входящих запросов

### Фаза 4 — Тренер и Админ ✅ готово (базовая версия)
- ✅ Экран ожидания подтверждения тренера
- ✅ Панель тренера: группы, приглашения, дашборд
- ✅ Панель Админа: заявки, подтвердить / отклонить
- [ ] Realtime: автопереход тренера при одобрении (без перезагрузки)

### Фаза 5 — Полировка ✅ готово
- ✅ Dark mode, mobile-first, empty states, GitHub Pages

### Исправленные баги
- ✅ `.single()` → `.maybeSingle()` (PGRST116 для новых пользователей)
- ✅ Вкладки не переключались (`hidden` класс не убирался)
- ✅ RLS рекурсия `group_members` ↔ `groups` (удалена `groups_member_select`)
- ✅ Email домен `.local` → `.app` (Supabase отвергал `.local`)
- ✅ Email provider был отключён (включён в Providers)

### Осталось сделать
- [ ] Realtime для экрана ожидания тренера
- [ ] GitHub Actions — ping Supabase раз в 3 дня (защита от паузы Free tier)

---

## 11. Зависимости и инструменты

| Инструмент | Версия | Зачем |
|---|---|---|
| `@supabase/supabase-js` | v2 (CDN) | Auth, БД, Realtime |
| DiceBear API | v7.x | аватарки без ключа и без хранения |
| Tabler Icons | v3.6 (CDN) | иконки |
| Google Fonts: Inter + Unbounded | — | типографика |

**Инициализация:**
```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
<script>
  const { createClient } = supabase;
  const sb = createClient('YOUR_SUPABASE_URL', 'YOUR_ANON_KEY');
  // ANON_KEY безопасно хранить в коде — данные защищены RLS политиками
</script>
```

---

## 12. Таблица рисков и решений

| Риск | Решение |
|---|---|
| Ник занят | Unique constraint в БД + debounce-проверка в реальном времени при вводе |
| PIN забыт | Аккаунт не восстанавливается — предупредить при регистрации |
| Дублирование тренировки за день | `UPSERT ON CONFLICT (user_id, date) DO UPDATE` |
| Тренер до подтверждения видит чужие данные | RLS фильтрует по `is_active = true` на уровне БД |
| Supabase Free пауза проекта (1 нед. неактивности) | GitHub Actions: GET-запрос к Supabase API раз в 3 дня |
| Admin UUID скомпрометирован | Роль admin проверяется RLS на уровне БД, не только в JS |
| Два пользователя выбрали одинаковый ник одновременно | Unique constraint поймает, `signUp` вернёт ошибку → показать сообщение |

---

## 13. Будущие улучшения (резервный план)

### Magic Link — восстановление аккаунта при утере PIN

Добавить опциональное поле email в профиль. Если пользователь добавил email — при потере PIN отправляем Magic Link для входа, затем позволяем сменить PIN.

```javascript
// Привязка email к существующей сессии (Supabase linkIdentity)
await sb.auth.updateUser({ email: 'user@example.com' });

// Восстановление через Magic Link
await sb.auth.signInWithOtp({ email: 'user@example.com' });
```

**Приоритет:** реализовать после запуска MVP, когда появятся реальные пользователи которым нужно восстановление.
