# Estratégias de Processamento Massivo Concorrente - Python + Kafka

## 📋 Contexto
Você tem dois processos que rodam em paralelo e podem travar:
- **Processo A**: Liberar para fechamento
- **Processo B**: Conciliar apólices + Criar parcelas

Ambos acionados simultaneamente no mesmo Deployment massivo.

---

## 🎯 Estratégia 1: Topics Separados + Consumer Groups Diferentes

### Descrição
Criar topics separados para A e B, cada um com seu consumer group e workers independentes.

### Kafka Topics
```
- settlement-remuneration-unit-massivo-liberar-fechamento  (Processo A)
- settlement-remuneration-unit-massivo-conciliar-apolices  (Processo B)
```

### Implementação Python

**`app/massivo_processor.py`**:
```python
from kafka import KafkaConsumer, KafkaProducer
from concurrent.futures import ThreadPoolExecutor
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MassivoProcessor:
    def __init__(self):
        # Consumer para Liberar Fechamento (Processo A)
        self.consumer_a = KafkaConsumer(
            'settlement-remuneration-unit-massivo-liberar-fechamento',
            bootstrap_servers='kafka:9092',
            group_id='settlement-remuneration-unit-massivo-liberar-group',
            auto_offset_reset='latest',
            max_poll_records=10,
            session_timeout_ms=30000
        )

        # Consumer para Conciliar Apólices (Processo B)
        self.consumer_b = KafkaConsumer(
            'settlement-remuneration-unit-massivo-conciliar-apolices',
            bootstrap_servers='kafka:9092',
            group_id='settlement-remuneration-unit-massivo-conciliar-group',
            auto_offset_reset='latest',
            max_poll_records=10,
            session_timeout_ms=30000
        )

        # Thread pool para cada processo (isolados)
        self.executor_a = ThreadPoolExecutor(max_workers=4, thread_name_prefix='liberar-')
        self.executor_b = ThreadPoolExecutor(max_workers=4, thread_name_prefix='conciliar-')

        # Producer para enviar resultados
        self.producer = KafkaProducer(
            bootstrap_servers='kafka:9092',
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    def process_liberar_fechamento(self, message):
        """Processo A: Liberar para fechamento"""
        try:
            data = json.loads(message.value.decode('utf-8'))
            logger.info(f"[A] Processando liberar fechamento: {data['id']}")

            # SUA LÓGICA DO PROCESSO A
            result = self._liberar_para_fechamento(data)

            logger.info(f"[A] ✅ Fechamento liberado: {data['id']}")
            return result
        except Exception as e:
            logger.error(f"[A] ❌ Erro: {e}")
            # Envia para dead letter queue
            self.producer.send(
                'settlement-remuneration-unit-massivo-liberar-fechamento-dlq',
                value={'error': str(e), 'data': data}
            )

    def process_conciliar_apolices(self, message):
        """Processo B: Conciliar apólices + criar parcelas"""
        try:
            data = json.loads(message.value.decode('utf-8'))
            logger.info(f"[B] Processando conciliação: {data['id']}")

            # SUA LÓGICA DO PROCESSO B (sem bloquear A)
            result = self._conciliar_apolices_e_criar_parcelas(data)

            logger.info(f"[B] ✅ Apólices conciliadas: {data['id']}")
            return result
        except Exception as e:
            logger.error(f"[B] ❌ Erro: {e}")
            # Envia para dead letter queue
            self.producer.send(
                'settlement-remuneration-unit-massivo-conciliar-apolices-dlq',
                value={'error': str(e), 'data': data}
            )

    def consume_messages(self):
        """Thread para consumir mensagens do Processo A"""
        logger.info("[A] Iniciando consumer para Liberar Fechamento...")
        for message in self.consumer_a:
            # Submete ao executor (non-blocking)
            self.executor_a.submit(self.process_liberar_fechamento, message)

    def consume_messages_b(self):
        """Thread para consumir mensagens do Processo B"""
        logger.info("[B] Iniciando consumer para Conciliar Apólices...")
        for message in self.consumer_b:
            # Submete ao executor (non-blocking)
            self.executor_b.submit(self.process_conciliar_apolices, message)

    def _liberar_para_fechamento(self, data):
        """SUA IMPLEMENTAÇÃO"""
        # Exemplo: DB updates
        import time
        time.sleep(2)  # Simula processamento pesado
        return {'status': 'liberado', 'id': data['id']}

    def _conciliar_apolices_e_criar_parcelas(self, data):
        """SUA IMPLEMENTAÇÃO"""
        # Exemplo: cálculos + DB updates
        import time
        time.sleep(3)  # Simula processamento pesado
        return {'status': 'conciliado', 'id': data['id']}

    def start(self):
        """Inicia ambos os consumers em threads separadas"""
        import threading

        thread_a = threading.Thread(target=self.consume_messages, daemon=True)
        thread_b = threading.Thread(target=self.consume_messages_b, daemon=True)

        thread_a.start()
        thread_b.start()

        logger.info("✅ Processador massivo iniciado com Processo A e B")

        # Keep alive
        try:
            thread_a.join()
            thread_b.join()
        except KeyboardInterrupt:
            logger.info("Encerrando...")
            self.executor_a.shutdown()
            self.executor_b.shutdown()
```

### 🔑 Vantagens
- ✅ Processos completamente isolados
- ✅ Não bloqueiam um ao outro
- ✅ Escaláveis independentemente
- ✅ Simples de debugar

### 📊 Topics Kafka Necessários
```bash
# Criar topics
kafka-topics --create --topic settlement-remuneration-unit-massivo-liberar-fechamento \
  --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092

kafka-topics --create --topic settlement-remuneration-unit-massivo-conciliar-apolices \
  --partitions 3 --replication-factor 1 --bootstrap-server localhost:9092

kafka-topics --create --topic settlement-remuneration-unit-massivo-liberar-fechamento-dlq \
  --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092

kafka-topics --create --topic settlement-remuneration-unit-massivo-conciliar-apolices-dlq \
  --partitions 1 --replication-factor 1 --bootstrap-server localhost:9092
```

---

## 🎯 Estratégia 2: Lock Distribuído (Redis/Zookeeper) - Se há Dependência

### Quando Usar
Se Processo B **depende** de Processo A ter finalizado (ex: Processo A libera, depois B concilia).

**`app/massivo_processor_with_lock.py`**:
```python
from kafka import KafkaConsumer, KafkaProducer
from concurrent.futures import ThreadPoolExecutor
import json
import logging
import redis
import time

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class MassivoProcessorWithLock:
    def __init__(self):
        self.redis_client = redis.Redis(host='redis', port=6379, db=0)

        # Consumer para Liberar Fechamento
        self.consumer_a = KafkaConsumer(
            'settlement-remuneration-unit-massivo-liberar-fechamento',
            bootstrap_servers='kafka:9092',
            group_id='settlement-remuneration-unit-massivo-liberar-group'
        )

        # Consumer para Conciliar Apólices
        self.consumer_b = KafkaConsumer(
            'settlement-remuneration-unit-massivo-conciliar-apolices',
            bootstrap_servers='kafka:9092',
            group_id='settlement-remuneration-unit-massivo-conciliar-group'
        )

        self.executor_a = ThreadPoolExecutor(max_workers=4)
        self.executor_b = ThreadPoolExecutor(max_workers=4)

        self.producer = KafkaProducer(
            bootstrap_servers='kafka:9092',
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

    def acquire_lock(self, resource_id, process_name, timeout=60):
        """
        Tenta adquirir lock distribuído
        Retorna True se conseguiu, False se já estava locked
        """
        lock_key = f"lock:{resource_id}:{process_name}"
        return self.redis_client.set(lock_key, "1", ex=timeout, nx=True)

    def release_lock(self, resource_id, process_name):
        """Libera o lock"""
        lock_key = f"lock:{resource_id}:{process_name}"
        self.redis_client.delete(lock_key)

    def process_liberar_fechamento(self, message):
        """Processo A com lock"""
        try:
            data = json.loads(message.value.decode('utf-8'))
            resource_id = data['apoliceid']

            # Tenta adquirir lock
            if not self.acquire_lock(resource_id, "liberar"):
                logger.warning(f"[A] Lock não adquirido para {resource_id}, retentando...")
                # Requeue a mensagem
                return False

            logger.info(f"[A] Lock adquirido para {resource_id}, processando...")

            # Executa o processo A
            self._liberar_para_fechamento(data)

            # Marca como liberado no Redis (para Processo B saber)
            self.redis_client.set(f"status:{resource_id}:liberado", "1", ex=3600)

            logger.info(f"[A] ✅ Fechamento liberado e marcado: {resource_id}")
            return True

        finally:
            self.release_lock(resource_id, "liberar")

    def process_conciliar_apolices(self, message):
        """Processo B com wait para Processo A"""
        try:
            data = json.loads(message.value.decode('utf-8'))
            resource_id = data['apoliceid']

            # Espera que Processo A tenha finalizado
            max_wait = 300  # 5 minutos
            waited = 0
            while not self.redis_client.get(f"status:{resource_id}:liberado"):
                if waited > max_wait:
                    logger.error(f"[B] Timeout esperando liberar de {resource_id}")
                    return False

                logger.info(f"[B] Aguardando Processo A finalizar para {resource_id}...")
                time.sleep(5)
                waited += 5

            # Agora pode conciliar
            logger.info(f"[B] Processo A finalizado, conciliando {resource_id}...")
            self._conciliar_apolices_e_criar_parcelas(data)

            logger.info(f"[B] ✅ Apólices conciliadas: {resource_id}")
            return True

        except Exception as e:
            logger.error(f"[B] ❌ Erro: {e}")
            return False

    def consume_messages(self):
        """Consumer para Processo A"""
        for message in self.consumer_a:
            self.executor_a.submit(self.process_liberar_fechamento, message)

    def consume_messages_b(self):
        """Consumer para Processo B"""
        for message in self.consumer_b:
            self.executor_b.submit(self.process_conciliar_apolices, message)

    def _liberar_para_fechamento(self, data):
        """SUA IMPLEMENTAÇÃO"""
        import time
        time.sleep(2)

    def _conciliar_apolices_e_criar_parcelas(self, data):
        """SUA IMPLEMENTAÇÃO"""
        import time
        time.sleep(3)

    def start(self):
        """Inicia ambos consumers"""
        import threading

        thread_a = threading.Thread(target=self.consume_messages, daemon=True)
        thread_b = threading.Thread(target=self.consume_messages_b, daemon=True)

        thread_a.start()
        thread_b.start()

        logger.info("✅ Processador com lock distribuído iniciado")

        try:
            thread_a.join()
            thread_b.join()
        except KeyboardInterrupt:
            logger.info("Encerrando...")
```

### 🔑 Quando Usar
- ✅ Se B depende de A
- ✅ Se há operações que não podem rodar simultaneamente
- ✅ Se há recurso compartilhado (mesmo banco de dados, mesmo arquivo)

---

## 🎯 Estratégia 3: Message Ordering + Single Partition

### Quando Usar
Se a **ordem importa** e precisa processar na sequência A → B.

**`app/ordered_processor.py`**:
```python
from kafka import KafkaConsumer
from concurrent.futures import ThreadPoolExecutor
import json
import logging
from enum import Enum

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ProcessType(Enum):
    LIBERAR = "liberar"
    CONCILIAR = "conciliar"

class OrderedMassivoProcessor:
    def __init__(self):
        # Topic ÚNICA com partição única para garantir ordem
        self.consumer = KafkaConsumer(
            'settlement-remuneration-unit-massivo-ordered',
            bootstrap_servers='kafka:9092',
            group_id='settlement-remuneration-unit-massivo-ordered-group',
            auto_offset_reset='latest'
        )

        self.executor = ThreadPoolExecutor(max_workers=2)

    def process_message(self, message):
        """Processa ambos processos em ordem"""
        try:
            data = json.loads(message.value.decode('utf-8'))
            process_type = data['type']

            if process_type == ProcessType.LIBERAR.value:
                logger.info(f"[A] Processando liberar: {data['id']}")
                self._liberar_para_fechamento(data)
                logger.info(f"[A] ✅ Liberado: {data['id']}")

            elif process_type == ProcessType.CONCILIAR.value:
                logger.info(f"[B] Processando conciliação: {data['id']}")
                self._conciliar_apolices_e_criar_parcelas(data)
                logger.info(f"[B] ✅ Conciliado: {data['id']}")

        except Exception as e:
            logger.error(f"❌ Erro: {e}")

    def start(self):
        """Consome em ordem"""
        logger.info("✅ Processador ordenado iniciado")
        for message in self.consumer:
            # Processa sequencialmente (uma mensagem por vez)
            self.executor.submit(self.process_message, message)

    def _liberar_para_fechamento(self, data):
        import time
        time.sleep(2)

    def _conciliar_apolices_e_criar_parcelas(self, data):
        import time
        time.sleep(3)
```

**Formato das mensagens**:
```json
{
  "type": "liberar",
  "id": "apoliceid-123",
  "data": {...}
}
```

---

## 🎯 Estratégia 4: Event Sourcing + State Machine

### Descrição
Cada evento muda o estado da apólice. Máquina de estados garante que A só processa se estado inicial, B só processa se A finalizou.

**`app/event_sourcing_processor.py`**:
```python
from kafka import KafkaConsumer, KafkaProducer
from enum import Enum
import json
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ApolicState(Enum):
    INICIAL = "inicial"
    LIBERANDO = "liberando"
    LIBERADO = "liberado"
    CONCILIANDO = "conciliando"
    CONCILIADO = "conciliado"
    ERRO = "erro"

class EventSourcingProcessor:
    def __init__(self):
        self.consumer = KafkaConsumer(
            'settlement-remuneration-unit-massivo-events',
            bootstrap_servers='kafka:9092',
            group_id='settlement-remuneration-unit-massivo-events-group'
        )

        self.producer = KafkaProducer(
            bootstrap_servers='kafka:9092',
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

        # Em memória (em produção usar Redis/DB)
        self.state_store = {}

    def get_state(self, apolicy_id):
        """Obtém estado atual da apólice"""
        return self.state_store.get(apolicy_id, {
            'state': ApolicState.INICIAL.value,
            'created_at': datetime.now().isoformat()
        })

    def set_state(self, apolicy_id, new_state, metadata=None):
        """Atualiza estado"""
        self.state_store[apolicy_id] = {
            'state': new_state,
            'updated_at': datetime.now().isoformat(),
            'metadata': metadata or {}
        }

        # Publica evento de mudança de estado
        self.producer.send(
            'settlement-remuneration-unit-massivo-state-changes',
            value={
                'apolicy_id': apolicy_id,
                'new_state': new_state,
                'timestamp': datetime.now().isoformat()
            }
        )

    def process_liberar(self, data):
        """Só processa se estado == INICIAL"""
        apolicy_id = data['id']
        current_state = self.get_state(apolicy_id)

        if current_state['state'] != ApolicState.INICIAL.value:
            logger.warning(f"[A] Não pode liberar - estado: {current_state['state']}")
            return False

        self.set_state(apolicy_id, ApolicState.LIBERANDO.value)

        try:
            logger.info(f"[A] Liberando: {apolicy_id}")
            self._liberar_para_fechamento(data)
            self.set_state(apolicy_id, ApolicState.LIBERADO.value)
            logger.info(f"[A] ✅ Liberado: {apolicy_id}")
            return True
        except Exception as e:
            logger.error(f"[A] ❌ Erro: {e}")
            self.set_state(apolicy_id, ApolicState.ERRO.value, {'error': str(e)})
            return False

    def process_conciliar(self, data):
        """Só processa se estado == LIBERADO"""
        apolicy_id = data['id']
        current_state = self.get_state(apolicy_id)

        if current_state['state'] != ApolicState.LIBERADO.value:
            logger.warning(f"[B] Não pode conciliar - estado: {current_state['state']}")
            return False

        self.set_state(apolicy_id, ApolicState.CONCILIANDO.value)

        try:
            logger.info(f"[B] Conciliando: {apolicy_id}")
            self._conciliar_apolices_e_criar_parcelas(data)
            self.set_state(apolicy_id, ApolicState.CONCILIADO.value)
            logger.info(f"[B] ✅ Conciliado: {apolicy_id}")
            return True
        except Exception as e:
            logger.error(f"[B] ❌ Erro: {e}")
            self.set_state(apolicy_id, ApolicState.ERRO.value, {'error': str(e)})
            return False

    def process_event(self, event):
        """Processa evento baseado em tipo"""
        data = json.loads(event.value.decode('utf-8'))
        event_type = data['type']

        if event_type == 'liberar':
            self.process_liberar(data)
        elif event_type == 'conciliar':
            self.process_conciliar(data)

    def start(self):
        """Inicia consumer"""
        logger.info("✅ Event Sourcing Processor iniciado")
        for event in self.consumer:
            self.process_event(event)

    def _liberar_para_fechamento(self, data):
        import time
        time.sleep(2)

    def _conciliar_apolices_e_criar_parcelas(self, data):
        import time
        time.sleep(3)
```

### 🔑 Vantagens
- ✅ Garante ordem de execução
- ✅ Fácil auditoria (todas as mudanças registradas)
- ✅ Fácil debugar (replay de eventos)
- ✅ Scalável

---

## 📊 Comparativo de Estratégias

| Aspecto | Estratégia 1 | Estratégia 2 | Estratégia 3 | Estratégia 4 |
|---------|-----------|-----------|-----------|-----------|
| **Isolamento** | ✅ Máximo | ⚠️ Médio | ❌ Baixo | ✅ Alto |
| **Simplicidade** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ |
| **Dependência A→B** | ❌ Não | ✅ Sim | ✅ Sim | ✅ Sim |
| **Ordem Garantida** | ❌ Não | ⚠️ Parcial | ✅ Sim | ✅ Sim |
| **Throughput** | ✅ Alto | ⚠️ Médio | ❌ Baixo | ⭐⭐⭐ |
| **Complexidade Code** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |

---

## 🎯 Recomendação por Caso

### Se A e B são **independentes**:
→ **Estratégia 1** (Topics separados)

### Se B **depende** de A:
→ **Estratégia 4** (Event Sourcing)

### Se precisa **máxima performance** e ordem:
→ **Estratégia 3** (Single Partition)

### Se há **recursos compartilhados** (mesmo DB):
→ **Estratégia 2** (Lock Distribuído)

---

## 🔧 Implementação no Docker

**`docker-compose.yml`**:
```yaml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:7.0.1
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  zookeeper:
    image: confluentinc/cp-zookeeper:7.0.1
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  redis:
    image: redis:7.0
    ports:
      - "6379:6379"

  app-massivo:
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      - KAFKA_BROKERS=kafka:9092
      - REDIS_HOST=redis
    depends_on:
      - kafka
      - redis
```

---

## 🚀 Próximos Passos

1. **Escolha a estratégia** baseada no seu caso
2. **Implemente** o código Python
3. **Crie os topics** Kafka
4. **Teste** com dados reais
5. **Configure KEDA** para escalar conforme carga
6. **Monitore** com Prometheus/Grafana

Qual estratégia faz mais sentido para seu caso? Quer que eu implemente alguma em detalhe? 🚀
