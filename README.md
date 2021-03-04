Я в основном напишу критику, проект в целом выполнен хорошо и описанные моменты ниже просто сделают ваш код лучше.

## По поводу особенностей проекта - 
1. Очень хорошо, что вы документируете README файлы и приводите в пример результаты исполнения определенных роутов. Было бы не лишним писать ровно такие же комментарии в самом коде см. [doctests](https://elixir-lang.org/getting-started/mix-otp/docs-tests-and-with.html). Это на деле помогло бы вам не только выдать внешний интерфейс, но и создать автотесты (по факту вручную вы выполняли тоже самое)
2. Вопросы вызывает выбранный алгоритм - Pbkdf2, [обратите внимание](https://medium.com/analytics-vidhya/password-hashing-pbkdf2-scrypt-bcrypt-and-argon2-e25aaf41598e)
3. Обратите внимание, что в эликсире есть свой встроеный (достаточно строгий) форматер, который поможет вам легко привыкнуть к некоторой чистоте оформления кода - `mix format`
4. В качестве рекомендаций - обратите внимание на [credo](https://github.com/rrrene/credo) и [dialyzer](https://github.com/jeremyjh/dialyxir), они помогут вам быстрее понять, как писать более красиво и правильно
5. Я понимаю, что на тесты вы не обращали внимание, но в таком случае лучше их вырезать из проекта, поскольку mix test выдает 13 ошибок
6. Обратите внимание на то, что `phoenix` фреймворк из коробки делит проект на бизнес-логику (работу с базой) и логику веб-интерфейса, не стоит их смешивать как в файлике (phx_task/auth/error_handler.ex), поскольку при дроблении на микросервисы, да и при поиске необходимых частей необходимо интуитивно понимать, где находится каждый блок
7. Обратите внимание на то, что библиотека guardian имеет свой блок настроек, очень важно понимать, что токен должен "заканчиваться" через время [более детально](https://hexdocs.pm/guardian/Guardian.Token.Jwt.html)

```elixir
# Настроить можно так в config.exs
config :your_app, YourApp.Guardian,
  # ...
  ttl: {1, :minute}

# Или вызвать напрямую - 
YourApp.Guardian.encode_and_sign(resource, %{}, ttl: {1, :minute})
```

## По поводу кода -
1. В целом все хорошо, следует обратить внимание на конструкцию with и cond (в случае с with желательно использовать else, поскольку иначе может произойти критическая ошибка) -

```elixir
# Условно говоря в коде можно было бы заменить вот такой блок из user_controller 

 current_user = Guardian.Plug.current_resource(conn)

 if current_user do
   case Auth.authorizate_for_change(current_user.id, id, password) do
     {:ok, user} ->
       with {:ok, _} <- Auth.delete_user(user) do
         # render(conn, "success.json", message: :user_was_deleted)
         render(conn, "success.json", user: user)
       end

     {:error, reason} ->
       render(conn, "error.json", reason: reason)
   end
 else
   render(conn, "error.json", reason: :no_authenticated)
 end

# На вот такой, он не слишком короче из-за обработки ошибки, 
# но читается несколько легче, да и обработку ошибки можно вынести в отдельную общую функцию

with %{id: user_id} <- Guardian.Plug.current_resource(conn),
     {:ok, user}    <- Auth.authorizate_for_change(user_id, id, password),
     {:ok, _}       <- Auth.delete_user(user) do
    render(conn, "success.json", user: user)
else
    e -> 
        case e do
            {:error, reason} -> render(conn, "error.json", reason: reason)
            _other           -> render(conn, "error.json", reason: :no_authenticated)
        end
end

# А в модуле - auth
 if current_id == user_id do
   case get_user(user_id) do
     {:ok, user} ->
       check_password(user, password)

     {:error, reason} ->
       {:error, reason}
   end
 else
   {:error, :no_permission_to_change}
 end

# Переделать так - 
with true <- current_id == user_id,
     {:ok, user} <- get_user(user_id) do
    check_password(user, password)
else
    e -> if e === false, do: {:error, :no_permission_to_change}, else: e
end

```

2. Немного странное использование Ecto, поскольку вам пришлось использовать блок try - rescue. Возможно вы испытывали различные конструкции, но на деле в этом нет необходимости, можно переделать примерно так -

```elixir
 try do
   user = Repo.get_by!(User, login: login)
   {:ok, user}
 rescue
   Ecto.NoResultsError ->
     {:error, :user_not_found}
 end
# ->
case Repo.get_by(User, login: login) do
    nil  -> {:error, :user_not_found}
    user -> {:ok, user}
end

```
3. Обратите внимание на то, что у модулей есть [атрибуты](https://elixirschool.com/en/lessons/basics/modules/#module-attributes), поэтому можно было бы их использовать в модели, например, так

```elixir
# ваш код - 
...
  schema "users" do
    field :login, :string
    field :name, :string
    field :password, :string, virtual: true
    field :password_hash, :string

    timestamps()
  end

  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:login, :name, :password])
    |> validate_required([:login, :name, :password])
    |> validate_length(:password, min: 6)
    |> unique_constraint(:login)
    |> generate_password_hash
  end
...
# поменять на следующий блок -
# В данном случае это может показаться оверхэдом, но редко когда модели имеют так мало полей и не имеют связей, 
# из-за чего управлять им может становиться сложно и такой трюк отлично поможет
# ~w(login name password)a == [:login, :name, :password]
# ~w()a == []
# ~w - говорит о том, что строка преобразуется в список, 
# а говорит о том, что список будет состоять из атомов
...
@required_fields ~w(login name password)a
@optional_fields ~w()a

schema "users" do
 field :login, :string
 field :name, :string
 field :password, :string, virtual: true
 field :password_hash, :string

 timestamps()
end

@doc false
def changeset(user, attrs) do
 user
 |> cast(attrs, @required_fields ++ @optional_fields)
 |> validate_required(@required_fields)
 |> validate_length(:password, min: 6)
 |> unique_constraint(:login)
 |> generate_password_hash
end
...
```
