## Carimbo de Maciez

Aplicativo desktop em Java Swing para classificar a maciez de cortes de carne por peso, indicando a categoria e o destino comercial ideal.

### O que o sistema faz
- recebe o peso do corte em quilos;
- valida entrada (números, vírgula/ponto decimal, valores negativos e vazios);
- classifica o corte em `Macia`, `Média` ou `Dura`;
- mostra o destino comercial sugerido;
- guarda um histórico dos últimos carimbos;
- exibe uma interface estilo bancada de controle de qualidade.

### Regras de classificação
- até 2,5 kg → Macia → Mercado de Exportação Premium / Steakhouses
- acima de 2,5 até 5,0 kg → Média → Varejo Tradicional / Supermercados
- acima de 5,0 kg → Dura → Mercado de Exportação / Processamento Industrial

### Estrutura do projeto
```text
.
├── pom.xml
├── README.md
└── src/
    └── main/
        └── java/
            └── com/
                └── example/
                    └── maciez/
                        └── MeatTendernessDisplayApp.java
```

### Como executar

Pré-requisitos:
- Java 17+
- Maven 3.9+

No terminal, na raiz do projeto:

```bash
mvn clean package
mvn exec:java -Dexec.mainClass=com.example.maciez.MeatTendernessDisplayApp
```

Ou, se preferir rodar diretamente pela classe compilada:

```bash
javac -d target/classes $(find src/main/java -name "*.java")
java -cp target/classes com.example.maciez.MeatTendernessDisplayApp
```

### Como funciona a lógica
A aplicação usa a classe `TendernessClassifier` para decidir a categoria do corte com base em limiares configurados:

```java
new TendernessClassifier(2.5, 5.0);
```

A classe `WeightParser` valida e converte a entrada do usuário para double, aceitando valores em formato brasileiro e americano (`2,5` ou `2.5`).

### Observações
- a interface é totalmente desktop, usando Swing;
- o projeto foi reorganizado para ficar fácil de entender, compilar e manter;
- a lógica de regra de negócio foi separada da interface gráfica para facilitar futuras mudanças.

### Desenvolvimento
Se você quiser evoluir o projeto, os pontos naturais de melhoria são:
- persistência do histórico em arquivo;
- exportação dos tickets em CSV;
- suporte a diferentes faixas por tipo de carne;
- testes unitários para o classificador.
