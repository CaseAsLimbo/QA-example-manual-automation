# Примеры написанных и доработанных мной тестов
## Сервис rating-api ("Дубинушка")
### Проверка логики создания комментариев
python
@pytest.mark.parametrize(
    'body,lecturer_n,response_status,aiohttp_response_status,achievement_id',
    [
        (  # тест логики выдачи ачивки за первый комментарий
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            0,
        ),
        (  # тест логики блокирующей выдачу ачивки за первый комментарий, если она уже есть у юзера
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # тест логики выдачи ачивки в случае неудачного get-запроса к серверу
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_500_INTERNAL_SERVER_ERROR,
            0,
        ),
        (
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (
            {
                "subject": "test1_subject",
                "text": "test text",
                "mark_kindness": -2,
                "mark_freebie": -2,
                "mark_clarity": -2,
            },
            1,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # bad mark
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 5,
                "mark_freebie": -2,
                "mark_clarity": 0,
            },
            2,
            status.HTTP_400_BAD_REQUEST,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # deleted lecturer
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": -2,
                "mark_clarity": 0,
            },
            3,
            status.HTTP_404_NOT_FOUND,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # Anonymous comment
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": -2,
                "mark_clarity": 0,
                "is_anonymous": True,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # NotAnonymous comment
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": -2,
                "mark_clarity": 0,
                "is_anonymous": False,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # Not provided anonymity
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": -2,
                "mark_clarity": 0,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # Bad anonymity
            {
                "subject": "test_subject",
                "text": "test text",
                "mark_kindness": 1,
                "mark_freebie": -2,
                "mark_clarity": 0,
                "is_anonymous": 'asd',
            },
            0,
            status.HTTP_422_UNPROCESSABLE_ENTITY,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # regex test
            {
                "subject": "test_subject",
                "text": """ABCDEFGHIJKLMNOPQRSTUVWXYZ
                        abcdefghijklmnopqrstuvwxyz.,!?-
                        абвгдежзийклмнопрстуфхцчшщъыьэюя1234567890
                        \"\'[]{}`~<>^@#№$%;:&*()+=\\/""",
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
                "is_anonymous": False,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # forbidden symbols
            {
                "subject": "test_subject",
                "text": """ABCDEFGHIJKLMNOPQRSTUVWXYZ
                        abcdefghijklmnopqrstuvwxyz.,!?-
                        абвгдежзийк☻☺☺лмнопрстуфхцчшщъыьэюя1234567890""",
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
                "is_anonymous": False,
            },
            0,
            status.HTTP_400_BAD_REQUEST,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # long comment
            {
                "subject": "test_subject",
                "text": 'a' * 3001,
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
                "is_anonymous": False,
            },
            0,
            status.HTTP_400_BAD_REQUEST,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
        (  # long comment but not that long
            {
                "subject": "test_subject",
                "text": 'a' * 3000,
                "mark_kindness": 1,
                "mark_freebie": 0,
                "mark_clarity": 0,
                "is_anonymous": False,
            },
            0,
            status.HTTP_200_OK,
            status.HTTP_200_OK,
            settings.FIRST_COMMENT_ACHIEVEMENT_ID,
        ),
    ],
)
def test_create_comment(
    client,
    dbsession,
    lecturers,
    authlib_user,
    mocker,
    body,
    lecturer_n,
    response_status,
    aiohttp_response_status,
    achievement_id,
):
    # url для проверки логики выдачи ачивок
    achive_get_url = settings.API_URL + f"example/{authlib_user.get('id'):}"
    achive_post_url = (
        settings.API_URL
        + f"example/{settings.FIRST_COMMENT_ACHIEVEMENT_ID}/example/{authlib_user.get('id'):}"
    )

    # мок aiohttp get- и post- запросов связанных с выдачей ачивки
    mock_aiohttp_session = aiohttp_mock(
        authlib_user_id=authlib_user.get("id"),
        aiohttp_response_status=aiohttp_response_status,
        achievement_id=achievement_id,
        get_url=achive_get_url,
        post_url=achive_post_url,
    )

    mocker.patch("aiohttp.ClientSession", return_value=mock_aiohttp_session)

    params = {"lecturer_id": lecturers[lecturer_n].id}
    post_response = client.post(url, json=body, params=params)

    assert post_response.status_code == response_status

    if response_status == status.HTTP_200_OK:
        comment = Comment.query(session=dbsession).filter(Comment.uuid == post_response.json()["uuid"]).one_or_none()
        assert comment is not None
        assert comment.review_status is ReviewStatus.PENDING

        # проверка корректной записи user_id и fullname при анонимных и не анонимных комментариях
        if body.get("is_anonymous") is not False:
            assert comment.user_id is None
            assert comment.user_fullname is None
        else:
            assert comment.user_id == authlib_user.get("id")
            assert comment.user_fullname == authlib_user.get("userdata")[0]["value"]

        if "create_ts" in body:
            assert comment.create_ts == datetime.datetime.fromisoformat(body["create_ts"]).replace(tzinfo=None)
        if "update_ts" in body:
            assert comment.update_ts == datetime.datetime.fromisoformat(body["update_ts"]).replace(tzinfo=None)

        user_comment = (
            LecturerUserComment.query(session=dbsession)
            .filter(LecturerUserComment.lecturer_id == lecturers[lecturer_n].id)
            .one_or_none()
        )
        assert user_comment is not None

        # Проверка логики ачивки
        check_get_response = mock_aiohttp_session.get
        check_post_response = mock_aiohttp_session.post

        if aiohttp_response_status == status.HTTP_200_OK:
            # Проверяем правильность заголовков и url get-запроса
            get_headers = {"Accept": "application/json"}
            try:
                check_get_response.assert_any_call(achive_get_url, headers=get_headers)
            except AssertionError as e:
                raise AssertionError(
                    f"Ожидался GET-запрос на {achive_get_url} c загловками {get_headers},"
                    f"но вызов, либо не состоялся, либо были переданы неверные заголовки."
                ) from e

            if achievement_id != settings.FIRST_COMMENT_ACHIEVEMENT_ID:
                # проверяем правильность заголовков и url post-запроса
                post_headers = {"Accept": "application/json", "Authorization": settings.ACHIEVEMENT_GIVE_TOKEN}
                try:
                    check_post_response.assert_any_await(achive_post_url, headers=post_headers)
                except AssertionError as e:
                    raise AssertionError(
                        f"Ожидался POST-запрос на {achive_post_url} c загловками {post_headers},"
                        f"но вызов, либо не состоялся, либо были переданы неверные заголовки."
                    )

            else:
                check_post_response.assert_not_awaited()
        else:
            check_post_response.assert_not_awaited()

---
### Проверка корректности импорта комментариев из более ранней версии приложения
python
@pytest.mark.parametrize(
    "body, total, response_status",
    [
        (
            {
                "comments": [
                    {
                        "subject": "string",
                        "text": "string",
                        "mark_kindness": 0,
                        "mark_freebie": 0,
                        "mark_clarity": 0,
                        "lecturer_id": 1,
                        "create_ts": "2026-05-25T11:41:26.777Z",
                        "update_ts": "2026-05-25T11:41:26.777Z",
                    },
                    {
                        "subject": "string",
                        "text": "string",
                        "mark_kindness": 0,
                        "mark_freebie": 0,
                        "mark_clarity": 0,
                        "lecturer_id": 2,
                        "create_ts": "2026-05-25T11:41:26.777Z",
                        "update_ts": "2026-05-25T11:41:26.777Z",
                    },
                ],
            },
            2,
            status.HTTP_200_OK,
        ),
        (
            {"comments": []},
            0,
            status.HTTP_200_OK,
        ),
        (
            {
                "comments": [
                    {
                        "subject": "string",
                        "text": "string",
                        "mark_kindness": 0,
                        "mark_freebie": 0,
                        "mark_clarity": 0,
                        "lecturer_id": 4,
                        "create_ts": "2026-05-25T11:41:26.777Z",
                        "update_ts": "2026-05-25T11:41:26.777Z",
                    },
                ],
            },
            1,
            status.HTTP_200_OK,
        ),
        (
            {
                "comments": [
                    {
                        "subdject": "string",
                        "text": "string",
                        "mark_kindness": 0,
                        "mark_freebie": 0,
                        "mark_clarity": 0,
                        "lecturer_id": "abc",
                    },
                ],
            },
            None,
            status.HTTP_422_UNPROCESSABLE_ENTITY,
        ),
    ],
)
def test_import_comments(client, dbsession, lecturers, body, total, response_status):
    response = client.post(f"{url}/import", json=body)

    assert response.status_code == response_status

    new_comments = response.json()
    print(new_comments)

    assert total == new_comments.get("total")

    if new_comments.get("total") and total > 0:
        for comment in new_comments.get("comments"):
            comment_from_db = Comment.query(session=dbsession).filter(Comment.uuid == comment.get("uuid")).one_or_none()
            assert comment_from_db is not None
---
## rental-backed (модуль аренды вещей в приложении)
### Проверка корректности сортировки item-ов(вещей)
python
@pytest.mark.parametrize(
    "item_n, order_by, order, response_status",
    [
        (None, None, "desc", status.HTTP_200_OK),
        (0, None, "desc", status.HTTP_200_OK),
        (None, "type_id", "desc", status.HTTP_200_OK),
        (1, "type_id", "desc", status.HTTP_200_OK),
        (None, "is_available", "desc", status.HTTP_200_OK),
        (2, "is_available", "desc", status.HTTP_200_OK),
    ],
)
def test_get_items_check_order(client, items_with_different_types, item_n, order_by, order, response_status):
    dict_of_params = {
        "type_id": items_with_different_types[item_n].type_id if item_n is not None else None,
        "order_by": order_by,
        "order": order,
    }
    query = {k: v for k, v in dict_of_params.items() if v is not None}
    response = client.get(url, params=query)
    assert response.status_code == response_status

    data = response.json()

    check_order_by = query.get("order_by") or "id"
    check_order = query.get("order") or "asc"

    key = lambda x: x[check_order_by]
    compare = (lambda x, y: x >= y) if check_order == "desc" else (lambda x, y: x <= y)
    assert all(compare(key(x), key(y)) for x, y in zip(data, data[1:]))
---
### Проверка правильности возвращаемых типов item-ов
python
@pytest.mark.parametrize(
    "item_n, order_by, order, is_available, response_status, expected_len",
    [
        (0, None, None, True, status.HTTP_200_OK, 1),
        (0, "id", None, True, status.HTTP_200_OK, 1),
        (0, "type_id", "asc", False, status.HTTP_200_OK, 1),
        (0, "is_available", "desc", False, status.HTTP_200_OK, 1),
        (0, None, None, None, status.HTTP_200_OK, 2),
        (1, "id", "asc", False, status.HTTP_200_OK, 1),
        (1, "type_id", "desc", True, status.HTTP_200_OK, 1),
        (1, "is_available", None, True, status.HTTP_200_OK, 1),
        (1, None, "asc", True, status.HTTP_200_OK, 1),
        (2, "id", "desc", True, status.HTTP_200_OK, 1),
        (2, "type_id", None, False, status.HTTP_200_OK, 1),
        (2, "is_available", "asc", False, status.HTTP_200_OK, 1),
        (2, None, "desc", None, status.HTTP_200_OK, 2),
        (-1, "id", None, True, status.HTTP_200_OK, 3),
        (-1, "type_id", "asc", False, status.HTTP_200_OK, 3),
        (-1, "is_available", "desc", None, status.HTTP_200_OK, 6),
        (-1, None, "desc", None, status.HTTP_200_OK, 6),
    ],
)
def test_get_items_by_various_filters(
    client, items_with_different_types, item_n, order_by, order, is_available, response_status, expected_len
):
    dict_of_params = {
        "type_id": items_with_different_types[item_n].type_id if item_n != -1 else None,
        "order_by": order_by,
        "order": order,
        "is_available": is_available if is_available is not None else None,
    }
    query = {k: v for k, v in dict_of_params.items() if v is not None}
    response = client.get(url, params=query)
    assert response.status_code == response_status

    data = response.json()
    assert len(data) == expected_len
