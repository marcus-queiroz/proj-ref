
### 📚 **Solução Completa para Ingestão Dinâmica de Arquivos no SSIS**

Vamos detalhar a solução passo a passo para **importar diversos tipos de arquivos** (CSV, Excel, JSON, Access e Posicional) utilizando o **SSIS**, incluindo a configuração do pacote e o código necessário. Essa abordagem cobre:

- **Importação dinâmica** de arquivos de múltiplas pastas.
- **Mapeamento de colunas dinamicamente** com base em metadados.
- **Suporte a diferentes tipos de arquivo**, incluindo arquivos posicionais (largura fixa).
- **Flexibilidade e escalabilidade** para adicionar novos arquivos sem modificar o pacote SSIS.

### 🔧 **Pré-requisitos e Configuração Inicial**

- Instale o **SSIS** (SQL Server Integration Services).
- Certifique-se de ter as bibliotecas necessárias para leitura de arquivos no seu ambiente de desenvolvimento:
    - `CsvHelper` para CSV.
    - `ExcelDataReader` para Excel.
    - `Newtonsoft.Json` para JSON.
    - `System.Data.OleDb` para Access.

### 📝 **Etapa 1: Criar a Tabela de Metadados no SQL Server**

A tabela `FileMetadata` armazenará informações sobre as pastas e o layout dos arquivos.

#### **Tabela `FileMetadata`:**
```sql
CREATE TABLE FileMetadata (
    FolderID INT PRIMARY KEY IDENTITY(1,1),
    FolderPath VARCHAR(500),
    FileType VARCHAR(50),           -- Ex: CSV, Excel, JSON, Access, Positional
    Delimiter VARCHAR(10),          -- Ex: ',', ';', '|', (para arquivos CSV)
    DestinationTable VARCHAR(100),  -- Nome da tabela de destino no SQL Server
    SchemaMapping VARCHAR(MAX),     -- JSON com o mapeamento de colunas ou layout para arquivos posicionais
    Processed BIT DEFAULT 0         -- Indicador de processamento
);

```


#### **Exemplo de Dados na Tabela `FileMetadata`:**
```sql
INSERT INTO FileMetadata (FolderPath, FileType, Delimiter, DestinationTable, SchemaMapping, Processed)
VALUES
('C:\Data\ClientesCSV', 'CSV', ',', 'dbo.Clientes', '{"Nome": "CustomerName", "Idade": "CustomerAge"}', 0),
('C:\Data\PedidosExcel', 'Excel', '', 'dbo.Pedidos', '{"PedidoID": "OrderId", "DataPedido": "OrderDate"}', 0),
('C:\Data\TransacoesJSON', 'JSON', '', 'dbo.Transacoes', '{"TransacaoID": "TransId"}', 0),
('C:\Data\AcessosMDB', 'Access', '', 'dbo.Acessos', '{"UserID": "UserId"}', 0),
('C:\Data\ClientesPosicional', 'Positional', '', 'dbo.Clientes', '[{"ColumnName": "CustomerID", "Start": 0, "Length": 10}, {"ColumnName": "CustomerName", "Start": 10, "Length": 30}, {"ColumnName": "CustomerAge", "Start": 40, "Length": 3}]', 0);

```


### 📝 **Etapa 2: Configurar o Pacote SSIS**

1. **Criar Variáveis SSIS**:
    
    - `User::FilePath` (String): Caminho completo do arquivo atual.
    - `User::FileType` (String): Tipo de arquivo (`CSV`, `Excel`, `JSON`, `Access`, `Positional`).
    - `User::Delimiter` (String): Delimitador (para arquivos CSV).
    - `User::DestinationTable` (String): Nome da tabela de destino.
    - `User::SchemaMapping` (String): Mapeamento de colunas em JSON.
2. **Foreach Loop Container**:
    
    - Adicione um **Foreach Loop Container** para iterar pelos registros da tabela `FileMetadata`.
    - Use uma **Execute SQL Task** para selecionar as pastas e carregar as variáveis SSIS.
    - Query de seleção:
        ```sql
SELECT FolderPath, FileType, Delimiter, DestinationTable, SchemaMapping
FROM FileMetadata
WHERE Processed = 0;
		```
        
3. **Script Component Source**:
    
    - Adicione um **Script Component Source** ao Data Flow Task.
    - Configure as colunas de saída dinamicamente, como `Column1`, `Column2`, etc.

### 📝 **Etapa 3: Implementar o Código C# no Script Component**

#### **Código C#:**
```csharp
using System;
using System.Data;
using System.Data.OleDb;
using System.IO;
using Newtonsoft.Json;
using CsvHelper;
using ExcelDataReader;
using System.Collections.Generic;
using System.Globalization;

public override void CreateNewOutputRows()
{
    string filePath = Variables.FilePath;
    string fileType = Variables.FileType;
    string delimiter = Variables.Delimiter;
    string schemaMappingJson = Variables.SchemaMapping;

    var schemaMapping = JsonConvert.DeserializeObject<List<ColumnLayout>>(schemaMappingJson);
    DataTable dataTable = new DataTable();

    switch (fileType.ToLower())
    {
        case "csv":
            dataTable = ReadCsvFile(filePath, delimiter);
            break;
        case "excel":
            dataTable = ReadExcelFile(filePath);
            break;
        case "json":
            dataTable = ReadJsonFile(filePath);
            break;
        case "access":
            dataTable = ReadAccessFile(filePath);
            break;
        case "positional":
            dataTable = ReadPositionalFile(filePath, schemaMapping);
            break;
        default:
            throw new Exception("Tipo de arquivo não suportado.");
    }

    foreach (DataRow row in dataTable.Rows)
    {
        Output0Buffer.AddRow();
        foreach (var column in schemaMapping)
        {
            Output0Buffer[column.ColumnName] = row[column.ColumnName]?.ToString();
        }
    }
}

public class ColumnLayout
{
    public string ColumnName { get; set; }
    public int Start { get; set; }
    public int Length { get; set; }
}

private DataTable ReadCsvFile(string filePath, string delimiter)
{
    var table = new DataTable();
    using (var reader = new StreamReader(filePath))
    using (var csv = new CsvReader(reader, new CsvHelper.Configuration.CsvConfiguration(CultureInfo.InvariantCulture)
    {
        Delimiter = delimiter
    }))
    {
        table.Load(new CsvDataReader(csv));
    }
    return table;
}

private DataTable ReadExcelFile(string filePath)
{
    using (var stream = File.Open(filePath, FileMode.Open, FileAccess.Read))
    using (var reader = ExcelReaderFactory.CreateReader(stream))
    {
        return reader.AsDataSet().Tables[0];
    }
}

private DataTable ReadJsonFile(string filePath)
{
    return JsonConvert.DeserializeObject<DataTable>(File.ReadAllText(filePath));
}

private DataTable ReadAccessFile(string filePath)
{
    var table = new DataTable();
    using (var conn = new OleDbConnection($"Provider=Microsoft.ACE.OLEDB.12.0;Data Source={filePath};"))
    {
        conn.Open();
        using (var adapter = new OleDbDataAdapter("SELECT * FROM [TableName]", conn))
        {
            adapter.Fill(table);
        }
    }
    return table;
}

private DataTable ReadPositionalFile(string filePath, List<ColumnLayout> schemaMapping)
{
    var table = new DataTable();
    foreach (var column in schemaMapping) table.Columns.Add(column.ColumnName);

    foreach (var line in File.ReadLines(filePath))
    {
        var row = table.NewRow();
        foreach (var column in schemaMapping)
        {
            row[column.ColumnName] = line.Substring(column.Start, column.Length).Trim();
        }
        table.Rows.Add(row);
    }
    return table;
}

```


### 📝 **Etapa 4: Configurar o OLE DB Destination**

- No **OLE DB Destination**, configure a propriedade **OpenRowset** para usar a expressão:
```sql
@[User::DestinationTable]
```
    

### 📝 **Etapa 5: Marcar Arquivo como Processado**

Adicione uma **Execute SQL Task** após a carga para atualizar a tabela `FileMetadata`:
```sql
UPDATE FileMetadata SET Processed = 1 WHERE FolderPath = ?;
```


### 📈 **Conclusão**

Esta solução oferece:

- **Flexibilidade** para lidar com múltiplos tipos de arquivos e schemas.
- **Reusabilidade** com um único Script Component para todos os tipos de arquivo.
- **Escalabilidade** para adicionar novas pastas e arquivos facilmente.
- **Robustez** com validação e logs de processamento.