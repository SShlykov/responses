Я в основном напишу критику, проект в целом выполнен хорошо и описанные моменты ниже просто сделают ваш код лучше.

## По поводу особенностей проекта - 
1. Не стоит использовать CapitalCamelCase, flatcase и kebab-case в названиях, поскольку из коробки elixir предоставляет некоторую банальную защиту (модули называются с большой буквы) можно, например, заблокировать возможность обращаться к другим модулям (т.е. запретить использовать большие буквы). Унифицированный вариант - snake_case.
2. Было бы не лишним писать документацию в самом коде см. [doctests](https://elixir-lang.org/getting-started/mix-otp/docs-tests-and-with.html). Это на деле помогло бы вам при исправлении и проверке функций, но и создать автотесты (по факту вручную вы выполняли тоже самое). Кроме того помогло бы понять, например, фронтендеру, как вы планируете использование вашего сервиса
3. В целом выбран неплохой алгоритм - Bcrypt, но можно лучше [обратите внимание](https://medium.com/analytics-vidhya/password-hashing-pbkdf2-scrypt-bcrypt-and-argon2-e25aaf41598e)
4. У вас все хорошо с форматированием, тем не менее лучше его использовать, чтобы код был правильно отформатирован - это сильно ускоряет чтение  - `mix format`
5. В качестве рекомендаций - обратите внимание на [credo](https://github.com/rrrene/credo) и [dialyzer](https://github.com/jeremyjh/dialyxir), они помогут вам быстрее понять, как писать более красиво и правильно
6. Я понимаю, что на тесты вы не обращали внимание, но в таком случае лучше их вырезать из проекта, поскольку mix test выдает 11 ошибок
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
1. В целом все хорошо, следует обратить внимание на конструкцию with и cond (в случае с with желательно использовать else, поскольку иначе может произойти критическая ошибка), кроме того для одного условия  -

```elixir
# Условно говоря в коде можно было бы заменить вот такой блок из accounts 

 with {:ok, user} <- get_by_login(login),
         do: verify_password(password, user)

# На вот такой, он не упадет и обработает ошибку 


with {:ok, user} <- get_by_login(login),
         do: verify_password(password, user), else: {:error, :cant_get}
```

2. Обратите внимание на то, что у модулей есть [атрибуты](https://elixirschool.com/en/lessons/basics/modules/#module-attributes), поэтому можно было бы их использовать в модели, например, так

```elixir
# ваш код - 
...
schema "users" do
   field :login, :string
   field :name, :string
   field :password_hash, :string

   field :password, :string, virtual: true

   timestamps()
end

@doc false
def changeset(%User{} = user, attrs) do
   user
   # Remove hash, add pw + pw confirmation
   |> cast(attrs, [:login, :password, :name])
   # Remove hash, add pw + pw confirmation
   |> validate_required([:login, :password, :name])
   # Check that password length is >= 8
   |> validate_length(:password, min: 6)
   |> unique_constraint(:login)
   |> put_password_hash
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
