from flask import Flask, request, jsonify, render_template_string
import sqlite3

app = Flask(__name__)

# Criar tabela no SQLite
def criar_tabela():
    conn = sqlite3.connect('planilha.db')
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS produtos (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            nome TEXT NOT NULL,
            preco_compra REAL NOT NULL,
            imposto REAL NOT NULL,
            frete REAL NOT NULL,
            preco_venda REAL NOT NULL,
            data_compra TEXT NOT NULL,
            valor_total REAL NOT NULL,
            lucro REAL NOT NULL
        )
    """)
    conn.commit()
    conn.close()

# Salvar produto no banco de dados
@app.route('/salvar-produto', methods=['POST'])
def salvar_produto():
    data = request.json
    conn = sqlite3.connect('planilha.db')
    cursor = conn.cursor()
    cursor.execute("""
        INSERT INTO produtos (nome, preco_compra, imposto, frete, preco_venda, data_compra, valor_total, lucro)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
    """, (data['nome'], data['preco_compra'], data['imposto'], data['frete'], data['preco_venda'], data['data_compra'], data['valor_total'], data['lucro']))
    conn.commit()
    conn.close()
    return jsonify({"status": "success"}), 200

# Listar produtos do banco de dados
@app.route('/listar-produtos', methods=['GET'])
def listar_produtos():
    conn = sqlite3.connect('planilha.db')
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM produtos")
    produtos = cursor.fetchall()
    conn.close()
    return jsonify(produtos), 200

# Página principal com HTML integrado
@app.route('/')
def index():
    html_content = """
    <!DOCTYPE html>
    <html lang="pt-BR">
    <head>
      <meta charset="UTF-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1.0" />
      <title>IA-PLANILHA</title>
      <style>
        body {
          font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
          background-color: #f5f7fa;
          margin: 0;
          padding: 20px;
          color: #333;
          line-height: 1.6;
        }
        h1 {
          text-align: center;
          color: #004080;
          margin-bottom: 30px;
        }
        .login-container {
          max-width: 400px;
          margin: 0 auto;
          padding: 20px;
          background-color: #fff;
          border-radius: 8px;
          box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
          text-align: center;
        }
        .login-container input {
          width: 100%;
          padding: 12px;
          margin: 10px 0;
          border: 1px solid #ccc;
          border-radius: 5px;
          font-size: 16px;
        }
        .login-btn {
          width: 100%;
          padding: 12px;
          background-color: #004080;
          color: white;
          border: none;
          border-radius: 5px;
          font-size: 16px;
          cursor: pointer;
          transition: background-color 0.3s ease;
        }
        .login-btn:hover {
          background-color: #003060;
        }
        .error-msg {
          color: red;
          margin-top: 10px;
        }
        .table-container {
          margin-top: 30px;
        }
        table {
          width: 100%;
          border-collapse: collapse;
          background-color: white;
          border-radius: 10px;
          overflow: hidden;
          box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
        }
        th, td {
          padding: 12px;
          border: 1px solid #ddd;
          text-align: center;
        }
        th {
          background-color: #004080;
          color: white;
        }
        td input {
          width: 100%;
          padding: 6px;
          border: 1px solid #ccc;
          border-radius: 4px;
          text-align: right;
        }
        .add-row-btn, .export-btn {
          display: inline-block;
          margin: 10px 0;
          padding: 10px 20px;
          background-color: #004080;
          color: white;
          border: none;
          border-radius: 5px;
          cursor: pointer;
          transition: background-color 0.3s ease;
        }
        .add-row-btn:hover, .export-btn:hover {
          background-color: #003060;
        }
        canvas {
          margin-top: 30px;
        }
      </style>
    </head>
    <body>

      <!-- Login -->
      <div class="login-container" id="loginForm">
        <h2>Login</h2>
        <input type="text" id="username" placeholder="Usuário" />
        <input type="password" id="password" placeholder="Senha" />
        <button class="login-btn" onclick="login()">Entrar</button>
        <div class="error-msg" id="errorMsg"></div>
      </div>

      <!-- Tabela -->
      <div class="table-container" id="tableContainer" style="display: none;">
        <h1>Planilha Inteligente de Precificação com IA</h1>
        <table id="priceTable">
          <thead>
            <tr>
              <th>Produto</th>
              <th>Preço de Compra (R$)</th>
              <th>Imposto (%)</th>
              <th>Frete (R$)</th>
              <th>Preço de Venda (R$)</th>
              <th>Data da Compra</th>
              <th>Valor Total (R$)</th>
              <th>Lucro (R$)</th>
              <th>Ação</th>
            </tr>
          </thead>
          <tbody></tbody>
        </table>
        <button class="add-row-btn" onclick="adicionarLinha()">Adicionar Produto</button>
        <button class="export-btn" onclick="exportarCSV()">Exportar CSV</button>
        <canvas id="profitChart" width="400" height="200"></canvas>
      </div>

      <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
      <script>
        function login() {
          const username = document.getElementById('username').value;
          const password = document.getElementById('password').value;

          if (username === "admin" && password === "1005") {
            document.getElementById('loginForm').style.display = 'none';
            document.getElementById('tableContainer').style.display = 'block';
            carregarProdutos();
          } else {
            document.getElementById('errorMsg').textContent = "Credenciais inválidas. Tente novamente.";
          }
        }

        function calcularValorTotal(row) {
          const precoCompra = parseFloat(row.querySelector('.preco-compra').value) || 0;
          const imposto = parseFloat(row.querySelector('.imposto').value) || 0;
          const frete = parseFloat(row.querySelector('.frete').value) || 0;
          const precoVenda = parseFloat(row.querySelector('.preco-venda').value) || 0;

          const total = precoCompra + frete + (precoCompra * (imposto / 100));
          const lucro = precoVenda - total;

          row.querySelector('.valor-total').value = isNaN(total) ? '' : total.toFixed(2);
          row.querySelector('.lucro').value = isNaN(lucro) ? '' : lucro.toFixed(2);

          salvarProduto(row);
        }

        function salvarProduto(row) {
          const nome = row.querySelector('.nome').value;
          const precoCompra = parseFloat(row.querySelector('.preco-compra').value) || 0;
          const imposto = parseFloat(row.querySelector('.imposto').value) || 0;
          const frete = parseFloat(row.querySelector('.frete').value) || 0;
          const precoVenda = parseFloat(row.querySelector('.preco-venda').value) || 0;
          const dataCompra = row.querySelector('.data-compra').value;
          const valorTotal = parseFloat(row.querySelector('.valor-total').value) || 0;
          const lucro = parseFloat(row.querySelector('.lucro').value) || 0;

          fetch('/salvar-produto', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({
              nome, preco_compra: precoCompra, imposto, frete, preco_venda: precoVenda,
              data_compra: dataCompra, valor_total: valorTotal, lucro
            })
          });
        }

        function adicionarLinha() {
          const tbody = document.querySelector('#priceTable tbody');
          const novaLinha = document.createElement('tr');
          novaLinha.innerHTML = `
            <td><input type="text" class="nome" /></td>
            <td><input type="number" step="0.01" class="preco-compra" /></td>
            <td><input type="number" step="0.01" class="imposto" /></td>
            <td><input type="number" step="0.01" class="frete" /></td>
            <td><input type="number" step="0.01" class="preco-venda" /></td>
            <td><input type="date" class="data-compra" /></td>
            <td><input type="text" class="valor-total" readonly /></td>
            <td><input type="text" class="lucro" readonly /></td>
            <td><button onclick="excluirLinha(this)">Excluir</button></td>
          `;
          novaLinha.querySelectorAll('input').forEach(input => {
            input.addEventListener('input', () => calcularValorTotal(novaLinha));
          });
          tbody.appendChild(novaLinha);
        }

        function excluirLinha(button) {
          const row = button.closest('tr');
          row.remove();
        }

        function exportarCSV() {
          const rows = document.querySelectorAll('#priceTable tbody tr');
          let csvContent = "data:text/csv;charset=utf-8,";

          rows.forEach(row => {
            const rowData = Array.from(row.querySelectorAll('input')).map(input => input.value);
            csvContent += rowData.join(",") + "\\n";
          });

          const encodedUri = encodeURI(csvContent);
          const link = document.createElement("a");
          link.setAttribute("href", encodedUri);
          link.setAttribute("download", "produtos.csv");
          document.body.appendChild(link);
          link.click();
        }

        function carregarProdutos() {
          fetch('/listar-produtos')
            .then(response => response.json())
            .then(produtos => {
              const tbody = document.querySelector('#priceTable tbody');
              tbody.innerHTML = '';
              produtos.forEach(produto => {
                const novaLinha = document.createElement('tr');
                novaLinha.innerHTML = `
                  <td><input type="text" class="nome" value="${produto[1]}" /></td>
                  <td><input type="number" step="0.01" class="preco-compra" value="${produto[2]}" /></td>
                  <td><input type="number" step="0.01" class="imposto" value="${produto[3]}" /></td>
                  <td><input type="number" step="0.01" class="frete" value="${produto[4]}" /></td>
                  <td><input type="number" step="0.01" class="preco-venda" value="${produto[5]}" /></td>
                  <td><input type="date" class="data-compra" value="${produto[6]}" /></td>
                  <td><input type="text" class="valor-total" value="${produto[7]}" readonly /></td>
                  <td><input type="text" class="lucro" value="${produto[8]}" readonly /></td>
                  <td><button onclick="excluirLinha(this)">Excluir</button></td>
                `;
                tbody.appendChild(novaLinha);
              });
              atualizarGrafico(produtos);
            });
        }

        function atualizarGrafico(produtos) {
          const ctx = document.getElementById('profitChart').getContext('2d');
          const profitChart = new Chart(ctx, {
            type: 'bar',
            data: {
              labels: produtos.map(p => p[1]),
              datasets: [{
                label: 'Lucro (R$)',
                data: produtos.map(p => p[8]),
                backgroundColor: '#004080',
                borderColor: '#003060',
                borderWidth: 1
              }]
            },
            options: {
              scales: {
                y: {
                  beginAtZero: true
                }
              }
            }
          });
        }
      </script>
    </body>
    </html>
    """
    return render_template_string(html_content)

if __name__ == '__main__':
    criar_tabela()
    app.run(debug=True)
