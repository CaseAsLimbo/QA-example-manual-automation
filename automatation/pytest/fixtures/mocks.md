# Примеры написанных мной моков и фикстур для подмены данных и реального взаимодействия
## rental-backend (сервис Цифрового проката)
### Пример фикстуры для создание itme-ов(вещей) для тестов
pyton
def items_with_different_types(dbsession):
    """Фикстура Item.

    .. note::
        Фикстура создает 6 item. Каждые 2 с одинаковым типом и разными значениями is_available.
    """

    item_types = [
        ItemType(name="Type1"),
        ItemType(name="Type2"),
        ItemType(name="Type3"),
    ]
    for item_type in item_types:
        dbsession.add(item_type)
    dbsession.commit()

    items = [
        Item(type_id=item_types[0].id, is_available=True),
        Item(type_id=item_types[0].id, is_available=False),
        Item(type_id=item_types[1].id, is_available=True),
        Item(type_id=item_types[1].id, is_available=False),
        Item(type_id=item_types[2].id, is_available=True),
        Item(type_id=item_types[2].id, is_available=False),
    ]
    for item in items:
        dbsession.add(item)
    dbsession.commit()
    yield items

## rating-api
### Пример мока пользователя для подмены аутентификации
python
@pytest.fixture
def authlib_user():
    """
    Данные о пользователе, возвращаемые сервисом auth.
    """
    return {...}


@pytest.fixture()
def authlib_mock(mocker):
    auth_mock = mocker.patch("auth_lib.fastapi.UnionAuth.__call__", autospec=True)
    return auth_mock


@pytest.fixture()
def user_mock(authlib_mock, authlib_user):
    authlib_mock.return_value = authlib_user
    return authlib_mock


@pytest.fixture
def client(mocker, user_mock):
    client = TestClient(app)
    return client
---
### Пример мока aiohttp запросов 
from unittest.mock import AsyncMock, MagicMock
...
import pytest





def create_response_mock(status=status.HTTP_200_OK, payload=None):
    """Вспомогательная функция, создающая мок-объекты-ответы типа aiohttp.ClientResponse."""
    mock_post_response = AsyncMock()
    mock_post_response.status = status
    mock_post_response.json = AsyncMock(return_value=payload or {})
    return mock_post_response


def create_ae_context_manager(mock_response):
    """
    Вспомогательная функция, создающая мок-объекты имитируютщие асинхронный контекстный
    менеджер возвращающий мок-объекты-ответы типа aiohttp.ClientResponse
    (для подмены async with session.get(...) as response: ...).
    """
    ctx = AsyncMock()
    ctx.__aenter__.return_value = mock_response
    return ctx


def aiohttp_mock(authlib_user_id, aiohttp_response_status, achievement_id, get_url, post_url):
    """Функция создающая мок-объект сессию типа aiohttp.ClientSession. Реализует моки get- и post- методов данного объекта."""

    # url для проверки логики выдачи ачивок
    achive_get_url = get_url
    achive_post_url = post_url

    # coздаем моки ответов aiohttp get- и post- запросов
    mock_get_response = create_response_mock(
        status=aiohttp_response_status,
        payload={
            "user_id": authlib_user_id,
            "achievement": [
                {
                    "id": achievement_id,
                }
            ],
        },
    )
    mock_post_response = create_response_mock(payload={})
    get_responses = {achive_get_url: create_ae_context_manager(mock_get_response)}
    post_responses = {achive_post_url: mock_post_response}

    # функции для side_effect моков get- и post- aiohttp запросов, если запрос был к не тому url, мок всегда вернет 404(не используется для проверки)
    def get_side_effect(url, *args, **kwargs):
        return get_responses.get(url, create_ae_context_manager(create_response_mock(status=status.HTTP_404_NOT_FOUND)))

    def post_side_effect(url, *args, **kwargs):
        return post_responses.get(url, (create_response_mock(status=status.HTTP_404_NOT_FOUND)))

    # создаем мок сессии aiohttp.ClientSession
    mock_aiohttp_session = AsyncMock()
    # для мока session.get(...) используем MagicMock вместо AsyncMock, потому что это синхронный метод
    mock_aiohttp_session.get = MagicMock(side_effect=get_side_effect)
    mock_aiohttp_session.post.side_effect = post_side_effect
    mock_aiohttp_session.__aenter__.return_value = mock_aiohttp_session

    return mock_aiohttp_session

...

*в тесте:*
...
    # мок aiohttp get- и post- запросов связанных с выдачей ачивки
    mock_aiohttp_session = aiohttp_mock(
        authlib_user_id=authlib_user.get("id"),
        aiohttp_response_status=aiohttp_response_status,
        achievement_id=achievement_id,
        get_url=achive_get_url,
        post_url=achive_post_url,
    )
...
