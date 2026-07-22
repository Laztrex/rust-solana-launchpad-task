## Подготовка окружения

У меня уже стояли:
- Rust 1.97.1 (актуальный)
- Node.js, yarn, npm

Установил Solana CLI и Anchor:

    sh -c "$(curl -sSfL https://release.anza.xyz/v2.3.0/install)"
    avm install 0.32.1
    avm use 0.32.1

### Первая проблема: cargo build-sbf ругается на lockfile v4

При первой попытке собрать программы через `anchor build` или `cargo build-sbf` получил:

    error: failed to parse lock file at: .../Cargo.lock
    Caused by: lock file version 4 requires `-Znext-lockfile-bump`

Решение: создал `~/.cargo/config.toml` с содержимым:

    [unstable]
    next-lockfile-bump = true
    edition-2024 = true

## Генерация ключей и обновление program ID

Сгенерировал ключевые пары:

    solana-keygen new -o target/deploy/sol_usd_oracle-keypair.json
    solana-keygen new -o target/deploy/token_minter-keypair.json

Получил публичные ключи:

- Oracle: `C4dYoNH7B2gGTAkCTdbFbviQRSjEYp6ZHAD7dhNZixR5`
- Minter: `83VyNBbwTJnVWKJQWhf8GzrsyzNgZ4RDwjrTQREv34UH`

Обновил их во всех местах. 

## Сборка программ – конфликт версий зависимостей

При запуске `anchor build` начались проблемы с крейтами, требующими edition2024 (например, `indexmap 2.14.0`, `zeroize_derive 1.5.0`, `unicode-segmentation 1.13.3`).

Встроенный Cargo их не понимал, даже с флагом.

Решение: добавил явные фиксации версий в `Cargo.toml` каждого контракта:

    indexmap = "=2.2.6"
    zeroize = "=1.7.0"
    zeroize_derive = "=1.4.2"
    unicode-segmentation = "=1.12.0"

Это заставило Cargo использовать совместимые версии.

Сборка прошла успешно. После сборки удалил зависимости для сборки *backend*.

## Деплой на localnet

Запустил валидатор:

    solana-test-validator

Убедился, что кошелёк имеет SOL (сделал `solana airdrop 100`).

Задеплоил контракты:

    solana program deploy target/deploy/sol_usd_oracle.so
    solana program deploy target/deploy/token_minter.so

Обе транзакции прошли успешно, program ID остались теми же (использованы те же keypair).

Инициализировал оракул и минтер:

    node scripts/init-local.js

Скрипт выкинул `ORACLE_STATE_PUBKEY` и подтвердил создание аккаунтов.

Запустил бэкенд (для localnet) и фронтенд, проверил минт – всё работало.

## Деплой на Devnet

Переключился на сеть:

    solana config set --url devnet

### Пополнение кошелька

Сначала попытался через CLI:

    solana airdrop 1

Не получилось:
```bash
Requesting airdrop of 1 SOL
Error: airdrop request failed. This can happen when the rate limit is reached.
```

Пополнил через UI в https://faucet.solana.com.

Деплой оракула прошёл.

При деплое минтера получил ошибку нехватки SOL для буфера.

Закрыл старый буфер, пополнил ещё через https://faucet.solana.com/

После этого деплой минтера завершился успешно.

## Инициализация на devnet

    RPC_URL=https://api.devnet.solana.com node scripts/init-local.js

Инициализация прошла без ошибок, получил `ORACLE_STATE_PUBKEY` для devnet:

    BGc27LiNW6qZCrLCrkxBDAnJFdtSf2AhQr2DzhiWZzdM

## Настройка бэкенда для devnet

В `backend/.env` я прописал:

    SOLANA_RPC_HTTP=https://api.devnet.solana.com
    SOLANA_RPC_WS=wss://api.devnet.solana.com
    ORACLE_PROGRAM_ID=C4dYoNH7B2gGTAkCTdbFbviQRSjEYp6ZHAD7dhNZixR5
    MINTER_PROGRAM_ID=83VyNBbwTJnVWKJQWhf8GzrsyzNgZ4RDwjrTQREv34UH
    ORACLE_STATE_PUBKEY=BGc27LiNW6qZCrLCrkxBDAnJFdtSf2AhQr2DzhiWZzdM

Запустил:

    cd backend
    SOLANA_RPC_HTTP=https://api.devnet.solana.com SOLANA_RPC_WS=wss://api.devnet.solana.com cargo run

Бэкенд поднялся, в логах пошли обновления цены.

## Запуск фронтенда и проблема с пополнением кошелька для минта

Фронтенд запустился на `http://localhost:3000`.

Нажал `Connect`, выбрал Phantom, но баланс кошелька был почти нулевым (после деплоя осталось меньше 0.1 SOL).

Попытался запросить airdrop:

    solana airdrop 1

Получил ошибку:

    airdrop request failed. This can happen when the rate limit is reached.

Веб-кран тоже отказал:

    You have exceeded the 2 airdrops limit in the past 8 hour(s).

Таким образом, на текущий момент не могу пополнить баланс для выполнения тестовой транзакции минта на Devnet, чтобы получить ссылку на транзакцию.

## Что сделано и что осталось

### Сделано

- Полностью настроено окружение.
- Сгенерированы свои ключи и обновлены все конфиги.
- Контракты собраны и задеплоены на localnet и devnet.
- Инициализация оракула и минтера на обеих сетях.
- Бэкенд работает на обеих сетях (обновляет цену и слушает события).
- Фронтенд запущен и готов к тестированию.
- Тесты проходят (`make test`).

 - После установки Backpack и импорта кошелька я успешно выполнил тестовый минт на Devnet.  
 - Транзакция подтверждена, токен создан.    
    Ссылка на транзакцию добавлена в README. 