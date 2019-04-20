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

## Referências

https://blog.classycode.com/over-the-air-updating-an-esp32-29f83ebbcca2