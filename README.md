local redis = require "resty.redis"
local uuid = require("resty.uuid")
local os = require("os")

-- Configuração do Redis (substitua com suas configurações)
local redis_host = "127.0.0.1"
local redis_port = 6379

-- Função para gerar a chave API
local function generate_api_key(user_id, validity)
    local key = uuid.new()
    local expiration = "permanent"
    local expiration_ts = nil

    if validity == "weekly" then
        expiration_ts = os.time() + 7 * 24 * 3600
        expiration = tostring(expiration_ts)
    elseif validity == "monthly" then
        expiration_ts = os.time() + 30 * 24 * 3600
        expiration = tostring(expiration_ts)
    end

    -- Conecta ao Redis
    local red = redis:new()
    red:set_timeout(1000)
    local ok, err = red:connect(redis_host, redis_port)
    if not ok then
        print("Erro ao conectar ao Redis: " .. err)
        return nil, "Falha ao conectar ao Redis"
    end

    -- Salva os dados da chave como um hash no Redis
    local ok, err = red:hmset("api_key:" .. key,
        "user_id", user_id,
        "expiration", expiration
    )
    if not ok then
        print("Erro ao salvar a chave no Redis: " .. err)
        red:close()
        return nil, "Falha ao salvar a chave no Redis"
    end

    local ok, err = red:close()
    if not ok then
        print("Erro ao fechar conexão Redis: " .. err)
    end

    return key, expiration_ts
end

-- Função para autenticar a chave
local function authenticate_key(api_key)
    local red = redis:new()
    red:set_timeout(1000)
    local ok, err = red:connect(redis_host, redis_port)
    if not ok then
        print("Erro ao conectar ao Redis: " .. err)
        return false, "Falha ao conectar ao Redis"
    end

    local key_data, err = red:hgetall("api_key:" .. api_key)
    if not key_data then
        red:close()
        return false, "API key inválida"
    end

    local expiration = key_data["expiration"]
    if expiration ~= "permanent" then
        local now = os.time()
        if now > tonumber(expiration) then
            red:close()
            return false, "API key expirada"
        end
    end

    red:close()
    return true, key_data["user_id"]
end

-- Função para exibir o menu principal
local function show_main_menu(user_id)
    print("\n--- Menu Principal ---")
    print("Bem-vindo, usuário " .. user_id .. "!")
    print("1. Opção 1: Visualizar informações")
    print("2. Opção 2: Editar perfil")
    print("3. Sair")
    print("Escolha uma opção: ")
end

-- Função para exibir o menu de autenticação
local function show_auth_menu()
    print("\n--- Menu de Autenticação ---")
    print("1. Gerar chave API")
    print("2. Entrar com chave API")
    print("3. Sair")
    print("Escolha uma opção: ")
end

-- Loop principal
while true do
    show_auth_menu()
    local choice = io.read()

    if choice == "1" then
        print("Digite o ID do usuário: ")
        local user_id = io.read()
        print("Tipo de chave (weekly, monthly, permanent): ")
        local validity = io.read()
        local key, expiration = generate_api_key(user_id, validity)
        if key then
            local expiration_str = (expiration == nil) and "permanente" or os.date("%Y-%m-%d %H:%M:%S", expiration)
            print("Chave gerada: " .. key .. ", Expira em: " .. expiration_str)
        else
            print("Erro ao gerar a chave.")
        end
    elseif choice == "2" then
        print("Digite a chave API: ")
        local api_key = io.read()
        local authenticated, user_id = authenticate_key(api_key)
        if authenticated then
            -- Menu Principal
            while true do
                show_main_menu(user_id)
                local main_choice = io.read()

                if main_choice == "1" then
                    -- Opção 1: Visualizar informações
                    print("Visualizando informações do usuário " .. user_id .. "...")
                elseif main_choice == "2" then
                    -- Opção 2: Editar perfil
                    print("Editando perfil do usuário " .. user_id .. "...")
                elseif main_choice == "3" then
                    print("Saindo do menu principal...")
                    break -- Volta para o menu de autenticação
                else
                    print("Opção inválida.")
                end
            end
        else
            print("Autenticação falhou: " .. user_id)
        end
    elseif choice == "3" then
        print("Saindo...")
        break
    else
        print("Opção inválida.")
    end
end
