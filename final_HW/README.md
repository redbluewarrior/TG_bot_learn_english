# ТГ-чат-бот «Обучалка английскому языку» по курсу»
## Структура:
### 1. Файл main.py содержит основную логику бота
#### 1.1 Функция отправляет количество шагов:
    @bot.message_handler(func=lambda message: message.text == Command.STEP_COUNT)
    def get_user_step(message):
        cid = message.chat.id
        user = session.query(Users).filter(Users.chat_id == cid).first()
        if user is not NoneType:
            bot.send_message(cid, f'Ваше количество шагов: {user.step}')
            userStep[user.chat_id] = user.step
        else:
            print("New user detected, who hasn't used \"/start\" yet")
            return 0

#### 1.2 Функция ответа на команду "/start":
    @bot.message_handler(commands=['start'])
    def add_user(message):
        cid = message.chat.id
        markup = types.ReplyKeyboardMarkup(row_width=2)
        if cid not in known_users:
            register_btn = types.KeyboardButton(Command.REGISTER)
            markup.add(register_btn)
            bot.send_message(
                            cid,
                            "Привет, незнакомец!\n"
                            "Давай учить английский, но сначала зарегистрируйся!",
                             reply_markup=markup
                             )
        else:
            create_cards_btn = types.KeyboardButton(Command.CREATE_CARD)
            markup.add(create_cards_btn)
            name = session.query(Users).filter(Users.chat_id == cid).first()
            bot.send_message(
                            message.chat.id,
                            f'Привет, {name.name}! Давай займёмся делом!\n'
                            f'Создай свою {name.step + 1}-ую карточку',
                            reply_markup=markup
                             )

#### 1.3 Функции регистрации пользователя:
    @bot.message_handler(func=lambda message: message.text == Command.REGISTER)
    def register_user(message):
        bot.set_state(message.chat.id, MyStates.name, message.chat.id)
        bot.send_message(message.chat.id, "Введите имя:")
    
    @bot.message_handler(state=MyStates.name)
    def register_name(message):
        bot.add_data(message.from_user.id, message.chat.id, name=message.text)
        bot.set_state(message.from_user.id, MyStates.surname, message.chat.id)
        bot.send_message(message.chat.id, "Введите фамилию:")
    
    @bot.message_handler(state=MyStates.surname)
    def register_surname(message):
        cid = message.chat.id
        bot.add_data(message.from_user.id, message.chat.id, surname=message.text)
        with bot.retrieve_data(message.from_user.id, message.chat.id) as data:
            name = data['name']
            surname = data['surname']
        user = Users(chat_id=cid, name=name, surname=surname)
        session_add(user)
        known_users.append(cid)
        bot.delete_state(message.from_user.id, message.chat.id)
        bot.send_message(message.chat.id, "Вы успешно зарегистрированы!")
        add_user(message)
        for word in session.query(Word_couples).all():
            couple = User_Word_couple(user_chat_id=cid, word_couple_id=word.id)
            session_add(couple)

#### 1.4 Функция создания учебной карточки:
    @bot.message_handler(func=lambda message: message.text == Command.CREATE_CARD)
    def create_cards(message):
        cid = message.chat.id
        markup = types.ReplyKeyboardMarkup(row_width=2)
        global buttons
        global words_text_reply
        words_text_reply = []
        buttons = []
        random_num = get_random_number(cid)
        if random_num != 0:
            word_db = (session.query(Word_couples)
                       .join(User_Word_couple.couple)
                       .filter(Word_couples.id == random_num)
                       .filter(User_Word_couple.user_chat_id == cid)
                       .first()
                       )
            # print(word_db)
            target_word = word_db.word_en
            translate = word_db.word_ru  
            target_word_btn = types.KeyboardButton(target_word)
            buttons.append(target_word_btn)
            others = [word.word_en for word in (session.query(Word_couples)
                                                .join(User_Word_couple.couple)
                                                .filter(User_Word_couple.user_chat_id == cid)
                                                .distinct()
                                                .all()
                                                )
                      ]  
            others.remove(target_word)
            random.shuffle(others)
            other_words_btns = [types.KeyboardButton(word) for word in others[:3]]
            words_text_reply.append(target_word)
            words_text_reply.extend(others[:3])
            buttons.extend(other_words_btns)
            random.shuffle(buttons)
    
            markup.add(*buttons)
    
            greeting = f"Выбери перевод слова:\n🇷🇺 {translate}"
            bot.send_message(message.chat.id, greeting, reply_markup=markup)
            bot.set_state(message.from_user.id, MyStates.target_word, message.chat.id)
            with bot.retrieve_data(message.from_user.id, message.chat.id) as data:
                data['target_word'] = target_word
                data['translate_word'] = translate
                data['other_words'] = others
        else:
            add_word_btn = types.KeyboardButton(Command.ADD_WORD)
            buttons.append(add_word_btn)
            markup.add(*buttons)
            bot.send_message(cid, f'Ваш словарь пуст 🙃 !\n'
                                  f'Пожалуйста добавьте новое слово!')
            start_add_word(message)

#### 1.5 Функция создания новой учебной карточки:
    @bot.message_handler(func=lambda message: message.text == Command.NEXT)
    def next_cards(message):
        create_cards(message)

#### 1.6 Функция удаление слова:
    @bot.message_handler(func=lambda message: message.text == Command.DELETE_WORD)
    def delete_word(message):
        cid = message.chat.id
        with bot.retrieve_data(message.from_user.id, message.chat.id) as data:
            want_to_delet_word = data['target_word'] # удалить из БД
        # print(want_to_delet_word)
        deleted_word_1 = (session.query(Word_couples)
                          .join(User_Word_couple.couple)
                          .filter(User_Word_couple.user_chat_id == cid)
                          .filter(Word_couples.word_en == want_to_delet_word)
                          .first()
                          )
        if deleted_word_1 is not None:
            deleted_word_2 = (session.query(User_Word_couple)
                              .filter(User_Word_couple.user_chat_id == cid)
                              .filter(User_Word_couple.word_couple_id == deleted_word_1.id)
                              .delete()
                              )
            session.commit()
            bot.send_message(cid, f'Слово #{deleted_word_1.id}:\n'
                                  f'EN: {deleted_word_1.word_en}\n'
                                  f'RU: {deleted_word_1.word_ru}\n'
                                  f'Успешно удалено! 👍'
                             )
        else:
            bot.send_message(cid, f'\"Чтобы удалить что-нибудь ненужное, '
                                  f'нужно сначала добавить что-нибудь ненужное, '
                                  f'а у нас словарь пустой!\"\n'
                                  f'© дядя Фёдор 🐷'
                             )

#### 1.7 Функции добавления слова:
    @bot.message_handler(func=lambda message: message.text == Command.ADD_WORD)
    def start_add_word(message):
        # cid = message.chat.id
        # userStep[cid] = 1
        bot.set_state(message.chat.id, MyStates.add_word_en, message.chat.id)
        bot.send_message(message.chat.id, "Введите слово на английском :")
          
    
    @bot.message_handler(state=MyStates.add_word_en)
    def add_word_en(message):
        bot.add_data(message.from_user.id, message.chat.id, word_en=message.text)
    
    
        bot.set_state(message.from_user.id, MyStates.add_word_ru, message.chat.id)
        bot.send_message(message.chat.id, "Введите перевод на русский:")
    
    
    @bot.message_handler(state=MyStates.add_word_ru)
    def add_word_ru(message):
        cid = message.chat.id
        bot.add_data(message.from_user.id, message.chat.id, word_ru=message.text)
    
        with bot.retrieve_data(message.from_user.id, message.chat.id) as data:
            word_en = data['word_en']
            word_ru = data['word_ru']
        word_couple_add = Word_couples(word_en=word_en, word_ru=word_ru)
        session_add(word_couple_add)
        word_couple = (session.query(Word_couples)
                       .filter(Word_couples.word_en == word_en)
                       .filter(Word_couples.word_ru == word_ru)
                       .first()
                       )
        user_couple = User_Word_couple(user_chat_id=cid, word_couple_id=word_couple.id)
        session_add(user_couple)
    
        bot.delete_state(message.from_user.id, message.chat.id)
    
        user_learn_words_count = session.query(User_Word_couple).filter(User_Word_couple.user_chat_id == cid).count()
        bot.send_message(message.chat.id, f"Вы новое слово добавили в свой словарь!!! 👍\n"
                                          f"EN: {word_en}\nRU: {message.text}\n"
                                          f"Количество слов, которые вы изучаете: {user_learn_words_count}")

#### 1.8 Функция ответов:
    @bot.message_handler(func=lambda message: message.text in words_text_reply)
    def message_reply(message):
        cid = message.chat.id
        buttons_reply = []
        text = message.text
        markup = types.ReplyKeyboardMarkup(row_width=2)
        with bot.retrieve_data(message.from_user.id, message.chat.id) as data:
            target_word = data['target_word']
            if text == target_word:
                hint = show_target(data)
                hint_text = ["Отлично!❤", hint]
                session.query(Users).filter(Users.chat_id == cid).update({'step': Users.step + 1})
                session.commit()
                next_btn = types.KeyboardButton(Command.NEXT)
                add_word_btn = types.KeyboardButton(Command.ADD_WORD)
                delete_word_btn = types.KeyboardButton(Command.DELETE_WORD)
                get_step_btn = types.KeyboardButton(Command.STEP_COUNT)
                buttons_reply.extend([next_btn, add_word_btn, delete_word_btn, get_step_btn])
                hint = show_hint(*hint_text)
            else:
                for btn in buttons:
                    if btn.text == text:
                        btn.text = text + '❌'
                        break
                markup.add(*buttons)
                hint = show_hint("Допущена ошибка! 😢",
                                 f"Попробуй ещё раз вспомнить слово 🇷🇺{data['translate_word']}")
        markup.add(*buttons_reply)
        bot.send_message(message.chat.id, hint, reply_markup=markup)

#### 1.9 Функция ответа на некорректные данные:
    @bot.message_handler(func=lambda message: True, content_types=['text'])
    def message_reply(message):
        cid = message.chat.id
        bot.send_message(cid, f'Некорректные данные! 😢\n'
                              f'Если вам нужно перезапустить бота, '
                              f'пожалуйста, воспользуйтесь командой /start !'
                         )

### 2. Файл DB.py создает базу данных и наполняет ее первичными данными
### 3. В файле models.py содержатся модели (структура БД)
### 4. Файл config.py содержаться константы 

## Инструкция по использованию:
1) Запускаем DB.py формируем базу данных и заполняем ее
2) Запускаем main.py
3) В консоли видим сообщение "Start telegram bot...", значит бот запущен
4) Для начала пройдем регистрацию ведем имя и фамилию
5) После сможем работать с карточками и изучать слова
6) Можем менять список доступных слов (добавлять и убирать), у каждого пользователя список слов индивидуальный
7) Также можем отслеживать число решенных карточек и число изучаемых слов
