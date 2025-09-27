from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service as ChromeService
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
import time
import requests
import sys

# ====================
# CONFIGURAÇÕES - ⚠️ AJUSTE AQUI
# ====================
LOGIN_URL = "https://ead.unieuro.edu.br/login/index.php"
PDF_URL = "https://ead.unieuro.edu.br/pluginfile.php/58764/mod_resource/content/1/globo.pdf"
HOME_URL = "https://ead.unieuro.edu.br/"  # Página inicial após login

# 🔒 Insira suas credenciais aqui:
# 1. ⚠️ SUBSTITUA SEU_USUARIO_AQUI PELA SUA MATRÍCULA/LOGIN
USERNAME = "06404527103".strip() 
# 2. ⚠️ SUBSTITUA SUA_SENHA_AQUI PELA SUA SENHA
PASSWORD = "06404527103".strip()

# ⛔ Coloque o caminho real do seu ChromeDriver
# 3. ⚠️ SUBSTITUA PELO CAMINHO CORRETO DO CHROMEDRIVER.EXE NO SEU PC
CHROMEDRIVER_PATH = "C:/Users/aluno/Desktop/chromedriver-win64/chromedriver-win64/chromedriver.exe"

# ✅ Controle se o navegador deve continuar aberto para uso manual
modo_interativo = True  # ← Altere para False se quiser que o navegador feche sozinho


# ====================
# SETUP DO DRIVER
# ====================
def setup_driver(headless=False):
    try:
        chrome_options = Options()
        if headless:
            chrome_options.add_argument("--headless")
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")

        # Configura o serviço do Chrome com o caminho que você forneceu
        service = ChromeService(executable_path=CHROMEDRIVER_PATH)
        driver = webdriver.Chrome(service=service, options=chrome_options)
        return driver
    except Exception as e:
        print(f" Erro ao iniciar o ChromeDriver.")
        print(f"Verifique se o caminho: '{CHROMEDRIVER_PATH}' está correto e se o Chrome/ChromeDriver estão atualizados.")
        print(f"Detalhe do erro: {e}")
        sys.exit(1)


# ====================
# LOGIN NA PLATAFORMA
# ====================
def login(driver):
    print(" Acessando página de login...")
    driver.get(LOGIN_URL)

    try:
        # Espera até que o campo de username esteja presente na página
        WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.ID, "username")))

        print(" Inserindo credenciais...")
        driver.find_element(By.ID, "username").send_keys(USERNAME)
        driver.find_element(By.ID, "password").send_keys(PASSWORD)
        driver.find_element(By.ID, "loginbtn").click()

        print(" Login realizado. Aguardando redirecionamento...")
        # Adicione uma verificação de sucesso de login, se necessário, ou apenas uma pausa
        time.sleep(5) 
        
        # O navegador agora deve estar na HOME_URL ou na página de erro.

    except Exception as e:
        print(f" Erro ao fazer login: {e}")
        # Não encerra aqui, para que test_pdf_access possa ser chamado
        return


# ====================
# ACESSA O PDF
# ====================
def test_pdf_access(driver):
    print(f"\n Acessando o PDF: {PDF_URL}")
    driver.get(PDF_URL)
    time.sleep(5) # Aguarda o carregamento da página

    # 1. Verifica no próprio navegador
    if "erro" in driver.page_source.lower() or "não permitido" in driver.page_source.lower():
        print("O navegador exibiu uma mensagem de erro ou acesso negado.")
        return False

    # 2. Verifica via HTTP usando os cookies de sessão (mais confiável)
    print(" Verificando via HTTP com cookies de sessão (simulando download)...")
    try:
        cookies = driver.get_cookies()
        session = requests.Session()

        # Transfere os cookies de sessão do Selenium para a sessão Requests
        for cookie in cookies:
            session.cookies.set(cookie['name'], cookie['value'])

        # Faz um HEAD request para verificar o status e o Content-Type sem baixar o arquivo todo
        response = session.head(PDF_URL, allow_redirects=True)

        content_type = response.headers.get("Content-Type", "").lower()
        if response.status_code == 200 and 'application/pdf' in content_type:
            print(" PDF acessível, Status 200 e Content-Type correto.")
            return True
        else:
            print(f" PDF não acessível via HTTP. Status: {response.status_code}, Tipo: {content_type}")
            return False

    except Exception as e:
        print(f" Erro ao verificar PDF via HTTP: {e}")
        return False


# ====================
# EXECUÇÃO PRINCIPAL
# ====================
def main():
    print("--- INICIANDO TESTE DE ACESSO AO PDF ---")
    driver = setup_driver(headless=False)

    try:
        # Tenta fazer o login
        login(driver) 
        
        # Tenta acessar o PDF e verifica o resultado
        sucesso = test_pdf_access(driver)

        if sucesso:
            print("\n TESTE CONCLUÍDO COM SUCESSO: O PDF é acessível após o login.")
        else:
            print("\n TESTE FALHOU: O PDF não é acessível ou inválido após o login.")

    finally:
        print("\n-------------------------------------")
        if modo_interativo:
            print(" Modo interativo ativado.")
            print(f" Redirecionando para a página inicial ({HOME_URL})...")
            driver.get(HOME_URL)
            print(" Navegador permanecerá aberto para uso manual.")
            print(" Feche o navegador manualmente ou pare o script no terminal (Ctrl+C).")
            # Loop para manter o script rodando e o navegador aberto
            try:
                while True:
                    time.sleep(1)
            except KeyboardInterrupt:
                print("\nScript interrompido pelo usuário.")
                driver.quit()
        else:
            print(" Encerrando navegador automaticamente.")
            driver.quit()


if __name__ == "__main__":
    main()
