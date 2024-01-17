# Почему вложенность это плохо

![Почему вложенность это плохо](cover.png)

> Автор: Евгений Кучерявый

Всем привет, на связи Евгений Кучерявый и сегодня я хочу обсудить с вами почему большая вложенность приводит к проблемам и как эти проблемы решать.

> Код в этой статье будет на Python, но принципы подходят для любого языка в [императивной парадигме программирования](/articles/chto-takoe-paradigma).

Давайте сразу перейдём к примеру:

```
#!/usr/bin/env python3
from validator.v1 import Validator
from auth import register


def handle_sign_up_form(username, password, email, phone):
    if Validator.check_email(email):
        if Validator.check_phone(phone):
            if Validator.check_username(username):
                if Validator.check_password(password):
                    register(username, password, email, phone)
                    return {
                        'status': '200',
                        'details': 'Success'
                    }
                else:
                    return {
                        'status': '400',
                        # This code is so hard to read, that I made a mistake in error message
                        'details': 'Invalid or taken username'
                    }
            else:
                return {
                    'status': '400',
                    'details': 'Invalid or taken username'
                }
        else:
            return {
                'status': '400',
                'details': 'Invalid phone'
            }
    else:
        return {
            'status': '400',
            'details': 'Invalid email'
        }

```

Здесь у нас обработка формы регистрации пользователя:

1. Получаем четыре введённых пользователем значения: `username`, `пароль`, `почту` и `телефон`.
2. Проверяем валидность введённых данных и либо выдаём ошибку, либо регистрируем пользователя и сообщаем, что всё ок.


Как именно происходит проверка для нас на данном этапе не важно — она есть, она вынесена в отдельный файл и уже хорошо. Пока это просто тестовый класс, который всегда возвращает `True`:

```
#!/usr/bin/env python3


class Validator:
    @staticmethod
    def check_username(value: str) -> bool:
        return True

    @staticmethod
    def check_password(value: str) -> bool:
        return True

    @staticmethod
    def check_phone(value: str) -> bool:
        return True

    @staticmethod
    def check_email(value: str) -> bool:
        return True

```

**Вернёмся к основной части.**

Несмотря на то, что метод по своей сути простой, разобраться в нём может быть непросто, потому что тут достаточно большое ветвление (конструкции `if-else`) и большая вложенность.

Читать такой код достаточно сложно, хотя тут всего 4 поля. Что будет, если полей станет в два раза больше, даже представить страшно.

## Как уменьшить вложенность кода

### Подход 1: Inversion (Инверсия)

Инверсия — это переворачивание условия. И опять сразу к примеру:

```
#!/usr/bin/env python3
from validator.v1 import Validator
from auth import register


def handle_sign_up_form(username, password, email, phone):
    if not Validator.check_email(email):
        return {
            'status': '400',
            'details': 'Invalid email'
        }

    if not Validator.check_phone(phone):
        return {
            'status': '400',
            'details': 'Invalid phone'
        }

    if not Validator.check_username(username):
        return {
            'status': '400',
            'details': 'Invalid or taken username'
        }

    if not Validator.check_password(password):
        return {
            'status': '400',
            'details': 'Invalid or week password'
        }

    register(username, password, email, phone)
    return {
        'status': '200',
        'details': 'Success'
    }

```

Здесь мы перевернули условие и блок внутри `if` у нас выполняется, если значение не прошло валидацию. Такой подход позволяет нам поместить в `if` оператор `return` и прервать работу метода, выдав сообщение об ошибке.

Хотя код не стал короче, мы всё равно избавились от лишней вложенности и повысили читаемость — теперь не нужно искать, какой `else` куда относится. IDE нам всё подсказывает, но всё равно приходится напрячься, чтобы понять что к чему.

Кроме того, инверсия также позволяет повысить производительность — мы прерываем работу метода как только понимаем, что какое-то из полей не прошло валидацию (хотя это не всегда полезно).

> Инверсия возможна и в циклах с помощью операторов `break` и `continue`.

### Подход 2: Extraction (Извлечение)

Извлечение — это по сути разбиение кода на несколько методов. И мы даже уже успели им воспользоваться, потому что валидация у нас происходит в другом файле. Но тут мы можем пойти ещё дальше — вынести всю валидацию и формирование ошибки в другой файл:

```#!/usr/bin/env python3
from validator.v2 import Validator
from auth import register


def handle_sign_up_form(username, password, email, phone):
    validation_result = Validator.validate((
        { 'value': username, 'type': 'username' },
        { 'value': password, 'type': 'password' },
        { 'value': email, 'type': 'email' },
        { 'value': phone, 'type': 'phone' },
    ))
    if not validation_result['success']:
        return {
            'status': '400',
            'details': {
                'message': 'Validation error',
                'errors': validation_result['error']
            }
        }

    register(username, password, email, phone)
    return {
        'status': '200',
        'details': 'Success'
    }

```

Мы вынесли всё, что касается валидации в другой файл, а здесь осталось только:

1. Передача аргументов в метод валидации.
2. Проверка результата.
3. Вывод ошибки.

Тут, кстати, мы изменили сам способ валидации. Теперь проверяются все поля и на основе проверки формируется список ошибок. Вот как это выглядит под капотом:

```
#!/usr/bin/env python3


class Validator:
    VALIDATORS = {
        'email': Validator.check_email,
        'phone': Validator.check_phone,
        'username': Validator.check_username,
        'password': Validator.check_password,
    }

    ERRORS = {
        'email': 'Invalid email',
        'phone': 'Invalid phone',
        'username': 'Invalid or taken username',
        'password': 'Invalid or week password',
    }

    @staticmethod
    def validate(fields: tuple) -> dict:
        errors = []

        for field in fields:
            validator = Validator.VALIDATORS[field_type]
            if validator(field['value'], field['type']):
                continue
            errors.append(Validator.ERRORS[field['type']])

        return {
            'success': len(errors) == 0,
            'errors': errors
        }

    @staticmethod
    def check_username(value: str) -> bool:
        return True

    @staticmethod
    def check_password(value: str) -> bool:
        return True

    @staticmethod
    def check_phone(value: str) -> bool:
        return True

    @staticmethod
    def check_email(value: str) -> bool:
        return True

```

Тут мы немного потеряли в производительности, потому что дожидаемся проверки всех полей. Но, с другой стороны, мы стали более дружелюбны к пользователю, потому что теперь мы можем сразу указать ему на все допущенные ошибки. Так пользователь сможет поправить всё разом, не отправляя миллион запросов с невалидными данными.

## Заключение

Краткий вывод из этой статьи звучит так:

> В великой вложенность великие печали. И кто умножает глубину, то приумножает скорбь. Поэтому нужно переворачивать условия и выносить код в разные методы, классы и файлы.

Стремитесь к уровню вложенности не более 4, хотя и это во многих случаях много.

---

> [GitHub-репозиторий с кодом](https://github.com/ithype/indentation)
> Видео о ветвлении и вложенности кода
			<iframe
				className='youtube-iframe'
				src='https://www.youtube.com/embed/mUWEy2K_l5o?si=GphteXMlIvRuvcOs'
				title='YouTube video player'
				frameBorder='0'
				allow='accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share'
				allowFullScreen
			></iframe>
