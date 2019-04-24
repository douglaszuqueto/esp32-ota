# ESP32 - OTA

## Ìndice

## Esquema de partições (em testes)

Dentre algumas discussões com o José(Zé) envolvendo **OTA**, foi 'descoberto' que os esquemas de partição disponíveis para o ESP32 na IDE do Arduino(integração [arduino-esp32](https://github.com/espressif/arduino-esp32)) não tinha disponível uma partição valiosa denominada *factory*!

Essa partição podemos dizer que pode ser considerada como *'padrão de fábrica'*, ou seja, é o último firmware gravado via cabo em seu esp32. Toda atualização utilizando OTA será gravado nas partições *app0* e *app1*.

Então no lugar de termos 2 partições ota0 e ota1 que o esquema padrão(default) nos trás, nos teremos 3 partições. Cada uma terá 1MB de armazenamento para seu firmware - é claro que com essa opção, se terá um custo! Você terá um espaço de armazenamneto menor para seu firmware como consequência.

Então imagine um cenário onde você utilize OTA e por algum motivo seu firmware bugou ou algo equivalente, portanto talvez sua última chance seria voltar para a partição factory! Nessa partição o Zé deu a ideia de deixar um firmare bem cru contendo apenas uma camada de OTA para aceitar novos updades.

Caso você fique sem comunicação alguma, pode-se adicionar até um push button para realizar esse rollback para factory - um caso bem real por sinal :)

Bem, chega de enrolação, abaixo seguem alguns passos e informações para poder ser adicionado essa nova partição ao seu ambiente.

* Esquema default

Esse é o schema que geralmente é utilizado durante as gravações de firmware. Portanto todas gravações serão sendo feitas e alternadas entre app0 e app1.

**Name**|**Type**|**SubType**|**Offset**|**Size**|**Flags**
:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
nvs|      data| nvs|     0x9000|  0x5000| 
otadata|  data| ota|     0xe000|  0x2000| 
app0|     app|  ota_0|   0x10000| 0x140000| 
app1|     app|  ota_1|   0x150000|0x140000| 
eeprom|   data| 0x99|    0x290000|0x1000| 
spiffs|   data| spiffs|  0x291000|0x16F000|

### Diagrama de funcionamento(referência em anexo)

Para ficar mais claro o fluxo de atualizações, acabei achando o diagrama(sem querer rsrs) onde esse fluxo é ilustrado.

![img](https://cdn-images-1.medium.com/max/800/1*s320_ezWr_0EUvkg5Y9Edg.png)

### Criando um novo schema de partição - Factory

* Passo 1

Crie o arquivo CSV no diretório conforme modelo descrito abaixo: **/home/dzuqueto/Arduino/hardware/espressif/esp32/tools/partitions/factory.csv**

**Name**|**Type**|**SubType**|**Offset**|**Size**|**Flags**
:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
nvs|      data| nvs|     0x9000|  0x5000| 
otadata|  data| ota|     0xe000|  0x2000| 
factory|app|factory|0x10000|1M| 
app0|     app|  ota_0|0x110000|1M| 
app1|     app|  ota_1|0x210000|1M| 

Caso deseje fazer o download do arquivo, segue o link: **[factory.csv](https://raw.githubusercontent.com/douglaszuqueto/esp32-ota/master/.github/factory.csv)**

Para mais detalhes sobre partições e particionamento, segue link da documentação: [Link](https://docs.espressif.com/projects/esp-idf/en/latest/api-guides/partition-tables.html)

**Obs:** O caminho do arquivo irá variar dependendo do seu sistema operacional!

* Passo 2

Para incluir a opção do novo schema criado(Factory), se faz necessário a edição do seguinte arquivo:  **/home/dzuqueto/Arduino/hardware/espressif/esp32/boards.txt**

Levando-se em consideração que a placa alvo seja o modelo **ESP32 Dev Module**, o que você verá no arquivo será algo como:

```
esp32.menu.PartitionScheme.default=Default
esp32.menu.PartitionScheme.default.build.partitions=default
esp32.menu.PartitionScheme.minimal=Minimal (2MB FLASH)
esp32.menu.PartitionScheme.minimal.build.partitions=minimal
esp32.menu.PartitionScheme.no_ota=No OTA (Large APP)
esp32.menu.PartitionScheme.no_ota.build.partitions=no_ota
esp32.menu.PartitionScheme.no_ota.upload.maximum_size=2097152
esp32.menu.PartitionScheme.huge_app=Huge APP (3MB No OTA)
esp32.menu.PartitionScheme.huge_app.build.partitions=huge_app
esp32.menu.PartitionScheme.huge_app.upload.maximum_size=3145728
esp32.menu.PartitionScheme.min_spiffs=Minimal SPIFFS (Large APPS with OTA)
esp32.menu.PartitionScheme.min_spiffs.build.partitions=min_spiffs
esp32.menu.PartitionScheme.min_spiffs.upload.maximum_size=1966080
esp32.menu.PartitionScheme.fatflash=16M Fat
esp32.menu.PartitionScheme.fatflash.build.partitions=ffat
```

Adicione as seguintes linhas:

```
esp32.menu.PartitionScheme.factory=Factory
esp32.menu.PartitionScheme.factory.build.partitions=factory
esp32.menu.PartitionScheme.factory.upload.maximum_size=1024000
```

* Resultado final na IDE

![img](https://raw.githubusercontent.com/douglaszuqueto/esp32-ota/master/.github/arduino-ide-v1.png)

* Passo 3: Alteração do arquivo *boot_app0.bin* (update)

Depois de ter feito vários testes, foi observado que o comportamento que o fluxo de atualização deveria seguir não estava correto. Foi ao que recorri ao Zé para saber um pouco mais sobre - se realmente o comportamento que estava acontecendo era de fato um problema!

Desde então começamos a fazer uma rotina de muitos testes para tentar encontrar o real problema que estava acontecendo.

Observe novamente o diagrama de fluxo abaixo: 
![img](https://cdn-images-1.medium.com/max/800/1*s320_ezWr_0EUvkg5Y9Edg.png)

O que eu havia mencionado la no inicio não estava 'acontecendo': "é o último firmware gravado via cabo em seu esp32.". O que deu para se entender é que, como a IDE não estava preparada para ter uma partição factory, alguns binários essenciais para o funcionamento do ESP32 também não estavam.

Então o que de fato acontecia? Com os partições geradas o upload era feito normalmente, ficando hospedado e rodando na partição factory. Quando era feita uma atualização OTA por meio de HTTP, o fluxo ia para app0 e dava boot na partição app0, tudo correto! O problema estava em fazer upload novamente via IDE do Arduino... o firmware era gravado normalmente, porém o boot iniciava em qual partição? app0 ou app1 kkk.

Só depois de inumeros testes comecei a entender os problemas e as peças começaram a se encaixar. No processo de build/upload 4 arquivos binários são gerados/utilizados e os mesmos são enviados para o ESP32, cada um no seu respectivo endereço de memória.

* Bootloader
* Otadata ( boot_app0.bin )
* Partitions
* App

Apenas 2 da lista acima eram gerados a cada build do firmware: Partitions e App - ambos ficam armazenados na pasta temporária do sistema juntamente com outros arquivos.

Bootloader e otadata sempre eram os mesmos. Foi ai que surgiu uma grande curiosidade e possível causa do problema, pois utilizando ESP-IDF, em todo build os 4 arquivos sempre eram gerados.

Então na rotina de testes, eu peguei os 2 arquivos(os fixos) gerados e realizei a substituição na pasta onde estavam armazenados, então ai eu consegui desmistificar o grande mistério. O grande vilão da jogada era o **boot_app0.bin**, como ele era fixo e não existia partição factory, certamente algo dentro dele se fazia essencial para o correto fluxo de atualizações.

Mas e agora, como resolver esse 'pequeno' problema?

Vamos la, os 2 arquivos mencionados ficam dentro da pasta onde suas ferramentas para usar esp32 estão instaladas.

* Bootloader - /home/dzuqueto/Arduino/hardware/espressif/esp32/tools/sdk/bin
* boot_app0 - /home/dzuqueto/Arduino/hardware/espressif/esp32/tools/partitions

O que precisa ser alterado é apenas o boot_app0. Segue um gabarito da pasta *partitions* para você se localizar melhor:

```
.
├── boot_app0.bin
├── boot_app0 (cópia).bin
├── boot_app0 (outra cópia).bin
├── default_16MB.csv
├── default_8MB.csv
├── default.bin
├── default.csv
├── factory.csv
├── ffat.csv
├── huge_app.csv
├── large_spiffs_16MB.csv
├── minimal.csv
├── min_spiffs.csv
├── no_ota.csv
└── partitions.bin
```

Pode ver que eu fiz algumas cópias de segurança para não perder o arquivo original. Caso você deseja efetuar todo esse processo segue o link de download do binário: [boot_app0.bin](https://raw.githubusercontent.com/douglaszuqueto/esp32-ota/master/.github/boot_app0.bin)

Depois de todas essas modificações, eu refiz todos os testes, inclusive com a partição padrão, e mesmo com a alteração do arquivo, tudo funcionou perfeitamente! Recomendo que teste o exemplo que deixei abaixo - [Blink OTA](https://github.com/douglaszuqueto/esp32-ota#blink---ota)

Eu acredito que, caso você faça todo esse processo tudo irá funcionar corretamente. Mas caso não ocorra, pode me chamar no telegram ou no [Grupo do FernandoK(Telegram)](https://t.me/fernandok_oficial) onde toda essa discussão se desenrolou.

## Blink

```c
#include "freertos/FreeRTOS.h"
#include "esp_ota_ops.h"

#define LED_BUILTIN 2

void setup() {
  pinMode(LED_BUILTIN, OUTPUT);
  Serial.begin(115200);
}

String getRunningPartition() {
  const esp_partition_t* partition = esp_ota_get_running_partition();
  Serial.printf("Partition: %s\n", partition->label);

  return partition->label;
}

void loop() {
  digitalWrite(LED_BUILTIN, HIGH);
  delay(1000);
  digitalWrite(LED_BUILTIN, LOW);
  delay(1000);

  getRunningPartition();
}
```

Com a função *esp_ota_get_running_partition()* você sempre saberá em que partição seu firmware estará rodando!

```c
String getRunningPartition() {
  const esp_partition_t* partition = esp_ota_get_running_partition();
  Serial.printf("Partition: %s\n", partition->label);

  return partition->label;
}
```

## Blink - OTA

Abaixo segue um exemplo de OTA utilizando a biblioteca que o [Zé](https://www.facebook.com/JoseMorais69) desenvolveu. Para mais detalhes acesse: [link](https://github.com/urbanze/esp32_gota)

Fica também uma referéncia ao post que ele fez no Grupo Arduino Brasil falando um pouco sobre sua biblioteca: [link](https://www.facebook.com/groups/arduino.br/permalink/2285458301493259/)

```c
#include <http.h>
#include <WiFi.h>
#include "esp_ota_ops.h"

#define LED_BUILTIN 2
#define WIFI_SSID ""
#define WIFI_PASSWORD ""

String getRunningPartition() {
  const esp_partition_t* partition = esp_ota_get_running_partition();
  Serial.printf("Partition: %s\n", partition->label);

  return partition->label;
}

void setup()
{
  Serial.begin(115200);
  pinMode(LED_BUILTIN, OUTPUT);

  WiFi.mode(WIFI_STA);
  WiFi.begin(WIFI_SSID, WIFI_PASSWORD);

  OTA_HTTP ota;
  ota.init();
}

void loop()
{
  digitalWrite(LED_BUILTIN, HIGH);
  delay(1000);
  digitalWrite(LED_BUILTIN, LOW);
  delay(1000);

  getRunningPartition();
}
```

## Referências

https://blog.classycode.com/over-the-air-updating-an-esp32-29f83ebbcca2