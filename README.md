# likecoin-akash
Setup and start Likecoin chain on akash 

Define environment variables before installing akash

```
AKASH_NET="https://raw.githubusercontent.com/ovrclk/net/master/mainnet"
AKASH_VERSION="$(curl -s "$AKASH_NET/version.txt")"
export AKASH_CHAIN_ID="$(curl -s "$AKASH_NET/chain-id.txt")"
export AKASH_NODE="$(curl -s "$AKASH_NET/rpc-nodes.txt" | shuf -n 1)"
```


Install akash (Linux)
```
curl https://raw.githubusercontent.com/ovrclk/akash/master/godownloader.sh | sh -s -- "v$AKASH_VERSION"
cp ./bin/akash /usr/local/bin/
```
![image](https://user-images.githubusercontent.com/59205554/135986794-63b597c8-3eac-41fe-a864-cb4a72ef5a52.png)


Create wallet
```
akash keys add default
- name: default
type: local
address: akash1ra5sxladp3wv5ej9p8qx5y227zhya8sfqrdw8h
pubkey: akashpub1addwnpepqtdc60d8yfayuq6340waga494w9uknm2y0jpl37zyzj8wx6fa2cmq06p39e
mnemonic: ""
threshold: 0
pubkeys: []

**Important** write this mnemonic phrase in a safe place.
It is the only way to recover your account if you ever forget your password.

wood ball walnut transfer tower soon into very spatial note grief cliff dismiss ability sun exist twin tower marine crazy design gate lift bulk
```
![image](https://user-images.githubusercontent.com/59205554/135986906-d855c6f3-73ac-4a21-a3fa-8839ddb289f0.png)

Save the mnemonic phrase, without it, wallet recovery will be impossible.

Register variables with the name and address of the wallet
```
export AKASH_ACCOUNT_ADDRESS="$(akash keys show default -a)"
export AKASH_KEY_NAME="default"
```
To continue, you need to purchase AKT tokens - https://akash.network/token

Check the balance
```
akash --node "$AKASH_NODE" query bank balances "$AKASH_ACCOUNT_ADDRESS"
```
![image](https://user-images.githubusercontent.com/59205554/135988194-aff8cad0-5888-47e9-97f0-53e80a235f59.png)

You need at lease 5.1 akt on balance, it is 5 100 000 uakt

Create certificate 
```
akash tx cert create client --from=$AKASH_KEY_NAME --chain-id $AKASH_CHAIN_ID --node $AKASH_NODE --fees 5000uakt -y
```
![image](https://user-images.githubusercontent.com/59205554/135988299-1df8cb8a-5f6c-48fd-afdb-15a79594bb5b.png)

At this point, the installation of akash is complete.

Expand our configuration with an image liked
Create a config file deploy.yml
```
cat > deploy.yml <<EOF
---
version: "2.0"

services:
  liked:
    image: bloqhub/liked-ssh:0.1
    expose:
      - port: 22656
        as: 22656
        proto: tcp
        to:
          - global: true
      - port: 2242
        as: 2242
        proto: tcp
        to:
          - global: true
      - port: 26657
        as: 80
        proto: tcp
        to:
          - global: true


profiles:
  compute:
    liked:
      resources:
        cpu:
          units: 0.1
        memory:
          size: 512Mi
        storage:
          size: 512Mi
  placement:
    akash:
      attributes:
        host: akash
      signedBy:
        anyOf:
          - "akash1365yvmc4s7awdyj3n2sav7xfx76adc6dnmlx63"
      pricing:
        liked:
          denom: uakt
          amount: 100

deployment:
  liked:
    akash:
      profile: liked
      count: 1

EOF
```
Attention! Address "akash1365yvmc4s7awdyj3n2sav7xfx76adc6dnmlx63" leave unchanged - this is the escrow account address.
The cpu, memory and size parameters in this configuration are set close to the minimum, in real use it is necessary
increase them

Expanding our configuration
```
akash tx deployment create deploy.yml --from $AKASH_KEY_NAME --node $AKASH_NODE --chain-id $AKASH_CHAIN_ID --fees 200uakt -b sync -y

{"height":"0","txhash":"05CBBC9ACBD1F51AF5B0D254E2BFD56B1ACA42819E5BABEEC5086947625E45D4","codespace":"","code":0,"data":"","raw_log":"[]","logs":[],"info":"","gas_wanted":"0","gas_used":"0","tx":
null,"timestamp":""}
```
check the status of your installation
```
akash q tx 05CBBC9ACBD1F51AF5B0D254E2BFD56B1ACA42819E5BABEEC5086947625E45D4 --node=$AKASH_NODE
"05CBBC9ACBD1F51AF5B0D254E2BFD56B1ACA42819E5BABEEC5086947625E45D4" получаем из вывода предыдущей команды
```
in the output of this command, we are interested in the dseq value
Defining variables
```
export AKASH_DSEQ=2252882
export AKASH_GSEQ=1
export AKASH_OSEQ=1
```
check the deployment status
```
akash query deployment get --owner $AKASH_ACCOUNT_ADDRESS --node $AKASH_NODE --dseq $AKASH_DSEQ
```
и определяем ставки которые можно использовать для нашей конфигурации
```
akash query market bid list --owner=$AKASH_ACCOUNT_ADDRESS --node $AKASH_NODE --dseq $AKASH_DSEQ
```
в списке возвращаемом командой выбираем провайдера
и присваиваем переменной его адрес
```
export AKASH_PROVIDER=akash14c4ng96vdle6tae8r4hc2w4ujwrshdddtuudk0
```
арендуем выбранного провайдера
```
akash tx market lease create --chain-id $AKASH_CHAIN_ID --node $AKASH_NODE --owner $AKASH_ACCOUNT_ADDRESS --dseq $AKASH_DSEQ --gseq $AKASH_GSEQ --oseq $AKASH_OSEQ --provider
$AKASH_PROVIDER --from $AKASH_KEY_NAME --fees 200uakt -y
```
спустя несколько секунд проверяем статус
```
akash query market lease list — owner $AKASH_ACCOUNT_ADDRESS — node $AKASH_NODE — dseq $AKASH_DSEQ
```
Загружаем манифест для нашего образа
```
akash provider send-manifest deploy.yml --node $AKASH_NODE --dseq $AKASH_DSEQ --provider $AKASH_PROVIDER --from $AKASH_KEY_NAME
```
И получаем данные доступа:
```
akash provider lease-status --node $AKASH_NODE --dseq $AKASH_DSEQ --from $AKASH_KEY_NAME --provider $AKASH_PROVIDER
```
Часть вывода,
```
{
"host": "cluster.provider-0.prod.ams1.akash.pub",
"port": 2242,
"externalPort": 31549,
"proto": "TCP",
"available": 1,
"name": "liked"
},
```
нас интересует host и externalPort.
В этом примере мы подключаемся к нашей ноде:
```
ssh root@cluster.provider-0.prod.ams1.akash.pub -p 31549
```
пароль задается при создании образа, после первого входа его необходимо сменить
Также можно просмотреть логи нашей ноды:
```
akash provider lease-logs --node "$AKASH_NODE" --dseq "$AKASH_DSEQ" --gseq "$AKASH_GSEQ" --oseq "$AKASH_OSEQ" --provider "$AKASH_PROVIDER" --from "$AKASH_KEY_NAME"
```

*Производим настройку валидатора*

Создаем ключи
```
liked init validator --chain-id likecoin-public-testnet-3
liked keys add validator
```
запоминаем полученную мнемоническую фразу   

Получаем genesis.json файл для нашей сети
```
curl https://gist.githubusercontent.com/nnkken/4a161c14e9dc03f412c36d11cdf7ea27/raw/9265c348c9f79b918d99aeee7f6c29b6b3bc449f/genesis.json -o /root/.liked/config/genesis.json
```
Прописываем seed в конфигурационном файле
```
sed -ie 's/seeds = ""/seeds = "c5e678f14219c1f161cb608aaeda37933d71695d@nnkken.dev:31801"/g' /root/.liked/config/config.toml
```
рестартуем ноду для применения изменений
```
reboot
```
подключаемся и создаем валидатора
```
ssh root@cluster.provider-0.prod.ams1.akash.pub -p 31549
```
```
liked tx staking create-validator \
--chain-id  "likecoin-public-testnet-3" \
--from  validator \
--moniker  "MONIKER" \
--pubkey  $(liked tendermint show-validator) \
--commission-max-rate  1.0 \
--commission-max-change-rate  1.0 \
--commission-rate 0.5 \
--min-self-delegation  1 \
--amount  90000000000000nanoekil
```
