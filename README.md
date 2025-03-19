# Bacbo-sinais-
Script python bacbo
#!/bin/bash

# Script de instalação automática do sistema de sinais Bac-Bo para Telegram
# Criado para: Edimilson Das Neves
# Bot: @LOANTEMPEBOT
# Grupo: Tempedabacbo

echo "==================================================="
echo "  Instalação do Sistema de Sinais Bac-Bo para Telegram"
echo "==================================================="
echo ""

# Verifica se está rodando como root
if [ "$EUID" -ne 0 ]; then
  echo "Este script precisa ser executado como root (use sudo)."
  exit 1
fi

# Cria diretório para o sistema
echo "Criando diretório para o sistema..."
mkdir -p /opt/bac-bo-bot
cd /opt/bac-bo-bot

# Atualiza o sistema e instala dependências
echo "Atualizando o sistema e instalando dependências..."
apt update && apt upgrade -y
apt install -y python3-pip python3-venv git chromium-browser chromium-chromedriver screen

# Instala bibliotecas Python
echo "Instalando bibliotecas Python..."
pip3 install python-telegram-bot selenium beautifulsoup4 requests webdriver-manager

# Cria o arquivo scraper.py
echo "Criando arquivo scraper.py..."
cat > /opt/bac-bo-bot/scraper.py << 'EOL'
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import time
import logging
import json
from datetime import datetime
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
from bs4 import BeautifulSoup

# Configuração de logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

class BacBoScraper:
    def __init__(self):
        self.url = "https://leon.bet/pt-br/live-casino/evolution/play/bac-bo"
        self.driver = None
        self.results_history = []
        self.last_result = None
        
    def setup_driver(self) :
        """Configura o driver do Selenium para Chrome headless."""
        try:
            chrome_options = Options()
            chrome_options.add_argument("--headless")
            chrome_options.add_argument("--no-sandbox")
            chrome_options.add_argument("--disable-dev-shm-usage")
            chrome_options.add_argument("--disable-gpu")
            chrome_options.add_argument("--window-size=1920,1080")
            
            service = Service(ChromeDriverManager().install())
            self.driver = webdriver.Chrome(service=service, options=chrome_options)
            logger.info("Driver do Chrome configurado com sucesso")
            return True
        except Exception as e:
            logger.error(f"Erro ao configurar o driver: {str(e)}")
            return False
    
    def navigate_to_game(self):
        """Navega até a página do jogo Bac-Bo."""
        try:
            logger.info(f"Navegando para {self.url}")
            self.driver.get(self.url)
            
            # Aguarda o carregamento da página
            WebDriverWait(self.driver, 30).until(
                EC.presence_of_element_located((By.TAG_NAME, "body"))
            )
            
            # Verifica se o jogo está disponível ou se há restrição regional
            if "Jogo indisponível" in self.driver.page_source:
                logger.error("Jogo indisponível na região atual")
                return False
                
            logger.info("Navegação bem-sucedida")
            return True
        except Exception as e:
            logger.error(f"Erro ao navegar para o jogo: {str(e)}")
            return False
    
    def extract_results(self):
        """Extrai os resultados do jogo da página."""
        try:
            # Aguarda o carregamento dos resultados
            WebDriverWait(self.driver, 30).until(
                EC.presence_of_element_located((By.CSS_SELECTOR, "table.results-history, div.results-history, div.game-history"))
            )
            
            # Captura o HTML da página
            html = self.driver.page_source
            soup = BeautifulSoup(html, 'html.parser')
            
            # Procura pela tabela ou div que contém o histórico de resultados
            # Nota: Os seletores exatos podem precisar ser ajustados após inspeção da página real
            history_container = soup.select_one("table.results-history, div.results-history, div.game-history")
            
            if not history_container:
                logger.warning("Não foi possível encontrar o histórico de resultados")
                return []
            
            # Extrai os resultados individuais
            # Isso precisará ser ajustado com base na estrutura real da página
            results = []
            result_elements = history_container.select("td.result, div.result, span.result")
            
            for element in result_elements:
                # Determina se o resultado é Jogador (P), Banqueiro (B) ou Empate (T)
                if "player" in element.get("class", []) or "blue" in element.get("class", []):
                    result = "P"  # Jogador (Player)
                elif "banker" in element.get("class", []) or "red" in element.get("class", []):
                    result = "B"  # Banqueiro (Banker)
                elif "tie" in element.get("class", []) or "yellow" in element.get("class", []):
                    result = "T"  # Empate (Tie)
                else:
                    # Tenta determinar pelo texto
                    text = element.get_text().strip().lower()
                    if "p" in text or "player" in text or "jogador" in text:
                        result = "P"
                    elif "b" in text or "banker" in text or "banqueiro" in text:
                        result = "B"
                    elif "t" in text or "tie" in text or "empate" in text:
                        result = "T"
                    else:
                        continue  # Pula se não conseguir determinar
                
                results.append(result)
            
            logger.info(f"Extraídos {len(results)} resultados")
            return results
        except Exception as e:
            logger.error(f"Erro ao extrair resultados: {str(e)}")
            return []
    
    def detect_new_result(self, current_results):
        """Detecta se há um novo resultado comparando com o histórico anterior."""
        if not self.results_history or not current_results:
            # Primeiro ciclo ou nenhum resultado atual
            self.results_history = current_results
            return None
        
        # Verifica se há mais resultados agora do que antes
        if len(current_results) > len(self.results_history):
            # O novo resultado é o primeiro da lista (mais recente)
            new_result = current_results[0]
            self.results_history = current_results
            self.last_result = new_result
            return new_result
        
        # Verifica se os resultados mudaram (mesmo que o número seja o mesmo)
        if current_results != self.results_history:
            # Assume que o primeiro resultado é o mais recente
            new_result = current_results[0]
            self.results_history = current_results
            self.last_result = new_result
            return new_result
        
        return None
    
    def format_signal_message(self, result):
        """Formata a mensagem de sinal para envio ao Telegram."""
        timestamp = datetime.now().strftime("%H:%M:%S")
        
        if result == "P":
            result_text = "JOGADOR (Player) 🔵"
        elif result == "B":
            result_text = "BANQUEIRO (Banker) 🔴"
        elif result == "T":
            result_text = "EMPATE (Tie) 🟡"
        else:
            result_text = f"RESULTADO DESCONHECIDO ({result})"
        
        message = f"""
🎲 <b>SINAL BAC-BO</b> 🎲
⏰ <b>Horário:</b> {timestamp}
🎯 <b>Resultado:</b> {result_text}
🏆 <b>Plataforma:</b> Leon Bet
        """
        
        return message.strip()
    
    def save_results(self):
        """Salva os resultados em um arquivo JSON."""
        try:
            data = {
                "last_updated": datetime.now().isoformat(),
                "results": self.results_history,
                "last_result": self.last_result
            }
            
            with open("results_history.json", "w") as f:
                json.dump(data, f, indent=4)
            
            logger.info("Resultados salvos com sucesso")
            return True
        except Exception as e:
            logger.error(f"Erro ao salvar resultados: {str(e)}")
            return False
    
    def load_results(self):
        """Carrega os resultados de um arquivo JSON."""
        try:
            with open("results_history.json", "r") as f:
                data = json.load(f)
            
            self.results_history = data.get("results", [])
            self.last_result = data.get("last_result")
            
            logger.info(f"Carregados {len(self.results_history)} resultados do histórico")
            return True
        except FileNotFoundError:
            logger.info("Arquivo de histórico não encontrado, iniciando novo histórico")
            return False
        except Exception as e:
            logger.error(f"Erro ao carregar resultados: {str(e)}")
            return False
    
    def close(self):
        """Fecha o driver do Selenium."""
        if self.driver:
            self.driver.quit()
            logger.info("Driver fechado")

def main():
    """Função principal para teste do scraper."""
    scraper = BacBoScraper()
    
    try:
        if not scraper.setup_driver():
            logger.error("Falha ao configurar o driver, encerrando")
            return
        
        if not scraper.navigate_to_game():
            logger.error("Falha ao navegar para o jogo, encerrando")
            scraper.close()
            return
        
        # Carrega resultados anteriores, se existirem
        scraper.load_results()
        
        # Extrai resultados atuais
        current_results = scraper.extract_results()
        
        if current_results:
            # Detecta novos resultados
            new_result = scraper.detect_new_result(current_results)
            
            if new_result:
                # Formata a mensagem de sinal
                message = scraper.format_signal_message(new_result)
                logger.info(f"Novo resultado detectado: {new_result}")
                logger.info(f"Mensagem formatada: {message}")
            else:
                logger.info("Nenhum novo resultado detectado")
        else:
            logger.warning("Nenhum resultado extraído")
        
        # Salva os resultados
        scraper.save_results()
    finally:
        # Garante que o driver seja fechado
        scraper.close()

if __name__ == "__main__":
    main()
EOL

# Cria o arquivo telegram_bot.py
echo "Criando arquivo telegram_bot.py..."
cat > /opt/bac-bo-bot/telegram_bot.py << 'EOL'
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import logging
import json
from telegram.ext import Application, CommandHandler, MessageHandler, filters

# Configuração de logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Token do bot fornecido pelo usuário
TOKEN = "7751342853:AAEtyuOtU6uZhJ1DFivovMa4zDpomB4QWw4"

# Arquivo para armazenar o ID do grupo
GROUP_ID_FILE = "group_id.json"

async def start(update, context):
    """Envia uma mensagem quando o comando /start é emitido."""
    await update.message.reply_text('Bot de sinais Bac-Bo iniciado! Enviarei automaticamente os resultados do jogo Bac-Bo da Leon Bet.')
    
    # Verifica se a mensagem veio de um grupo
    chat_id = update.effective_chat.id
    chat_type = update.effective_chat.type
    
    if chat_type in ['group', 'supergroup']:
        # Salva o ID do grupo
        save_group_id(chat_id)
        await update.message.reply_text(f'Este grupo foi configurado para receber sinais do Bac-Bo! ID do grupo: {chat_id}')

async def help_command(update, context):
    """Envia uma mensagem quando o comando /help é emitido."""
    await update.message.reply_text('Este bot envia automaticamente os resultados do jogo Bac-Bo da Leon Bet.')

async def get_chat_id(update, context):
    """Obtém e exibe o ID do chat atual."""
    chat_id = update.effective_chat.id
    chat_type = update.effective_chat.type
    
    await update.message.reply_text(f'O ID deste chat é: {chat_id} (Tipo: {chat_type})')
    
    if chat_type in ['group', 'supergroup']:
        # Salva o ID do grupo
        save_group_id(chat_id)
        await update.message.reply_text('Este grupo foi configurado para receber sinais do Bac-Bo!')
    
    return chat_id

async def handle_message(update, context):
    """Manipula mensagens recebidas e captura o ID do grupo."""
    # Verifica se a mensagem veio de um grupo
    chat_id = update.effective_chat.id
    chat_type = update.effective_chat.type
    
    if chat_type in ['group', 'supergroup']:
        # Salva o ID do grupo
        save_group_id(chat_id)
        await update.message.reply_text(f'Este grupo foi configurado para receber sinais do Bac-Bo! ID do grupo: {chat_id}')

async def send_test_message(update, context):
    """Envia uma mensagem de teste para o grupo."""
    group_id = load_group_id()
    
    if group_id:
        try:
            await context.bot.send_message(chat_id=group_id, text="Mensagem de teste do bot de sinais Bac-Bo!")
            await update.message.reply_text(f'Mensagem de teste enviada para o grupo com ID: {group_id}')
        except Exception as e:
            await update.message.reply_text(f'Erro ao enviar mensagem: {str(e)}')
    else:
        await update.message.reply_text('ID do grupo não configurado. Envie qualquer mensagem no grupo para configurar automaticamente.')

def save_group_id(group_id):
    """Salva o ID do grupo em um arquivo JSON."""
    try:
        data = {"group_id": group_id}
        with open(GROUP_ID_FILE, "w") as f:
            json.dump(data, f)
        logger.info(f"ID do grupo salvo: {group_id}")
        return True
    except Exception as e:
        logger.error(f"Erro ao salvar ID do grupo: {str(e)}")
        return False

def load_group_id():
    """Carrega o ID do grupo do arquivo JSON."""
    try:
        with open(GROUP_ID_FILE, "r") as f:
            data = json.load(f)
        group_id = data.get("group_id")
        logger.info(f"ID do grupo carregado: {group_id}")
        return group_id
    except FileNotFoundError:
        logger.info("Arquivo de ID do grupo não encontrado")
        return None
    except Exception as e:
        logger.error(f"Erro ao carregar ID do grupo: {str(e)}")
        return None

def main():
    """Inicia o bot."""
    # Cria a aplicação
    application = Application.builder().token(TOKEN).build()

    # Registra comandos
    application.add_handler(CommandHandler("start", start))
    application.add_handler(CommandHandler("help", help_command))
    application.add_handler(CommandHandler("getchatid", get_chat_id))
    application.add_handler(CommandHandler("test", send_test_message))
    
    # Manipulador para todas as mensagens
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

    # Inicia o bot
    application.run_polling()
    logger.info("Bot iniciado!")

if __name__ == '__main__':
    main()
EOL

# Cria o arquivo main.py
echo "Criando arquivo main.py..."
cat > /opt/bac-bo-bot/main.py << 'EOL'
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

import time
import logging
import asyncio
import json
from datetime import datetime
from telegram.ext import Application
from scraper import BacBoScraper

# Configuração de logging
logging.basicConfig(
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    level=logging.INFO
)
logger = logging.getLogger(__name__)

# Token do bot fornecido pelo usuário
TOKEN = "7751342853:AAEtyuOtU6uZhJ1DFivovMa4zDpomB4QWw4"

# Arquivo para carregar o ID do grupo
GROUP_ID_FILE = "group_id.json"

# Configurações
CHECK_INTERVAL = 30  # Intervalo em segundos para verificar novos resultados
MAX_RETRIES = 3  # Número máximo de tentativas em caso de falha

def load_group_id():
    """Carrega o ID do grupo do arquivo JSON."""
    try:
        with open(GROUP_ID_FILE, "r") as f:
            data = json.load(f)
        group_id = data.get("group_id")
        logger.info(f"ID do grupo carregado: {group_id}")
        return group_id
    except FileNotFoundError:
        logger.info("Arquivo de ID do grupo não encontrado")
        return None
    except Exception as e:
        logger.error(f"Erro ao carregar ID do grupo: {str(e)}")
        return None

async def send_signal_to_group(bot, message):
    """Função para enviar sinais para o grupo."""
    group_id = load_group_id()
    
    if group_id:
        try:
            await bot.send_message(chat_id=group_id, text=message, parse_mode='HTML')
            logger.info(f"Sinal enviado para o grupo {group_id}")
            return True
        except Exception as e:
            logger.error(f"Erro ao enviar sinal: {str(e)}")
            return False
    else:
        logger.error("ID do grupo não configurado. Envie uma mensagem no grupo para configurar automaticamente.")
        return False

async def monitor_game_results(bot):
    """Monitora os resultados do jogo e envia sinais quando detecta novos resultados."""
    scraper = BacBoScraper()
    retry_count = 0
    
    try:
        # Configura o driver
        if not scraper.setup_driver():
            logger.error("Falha ao configurar o driver, encerrando monitoramento")
            return
        
        # Navega para o jogo
        if not scraper.navigate_to_game():
            logger.error("Falha ao navegar para o jogo, encerrando monitoramento")
            scraper.close()
            return
        
        # Carrega resultados anteriores, se existirem
        scraper.load_results()
        
        # Aguarda até que o ID do grupo seja configurado
        while True:
            group_id = load_group_id()
            if group_id:
                break
            logger.info("Aguardando configuração do ID do grupo... Envie uma mensagem no grupo para configurar automaticamente.")
            await asyncio.sleep(10)
        
        # Envia mensagem inicial para o grupo
        await send_signal_to_group(bot, """
🎲 <b>BOT DE SINAIS BAC-BO INICIADO</b> 🎲
⏰ <b>Horário:</b> {}
🏆 <b>Plataforma:</b> Leon Bet

O bot está monitorando os resultados do jogo Bac-Bo e enviará sinais automaticamente.
        """.format(datetime.now().strftime("%H:%M:%S")))
        
        # Loop principal de monitoramento
        while True:
            try:
                # Extrai resultados atuais
                current_results = scraper.extract_results()
                
                if current_results:
                    # Detecta novos resultados
                    new_result = scraper.detect_new_result(current_results)
                    
                    if new_result:
                        # Formata a mensagem de sinal
                        message = scraper.format_signal_message(new_result)
                        logger.info(f"Novo resultado detectado: {new_result}")
                        
                        # Envia o sinal para o grupo
                        await send_signal_to_group(bot, message)
                    else:
                        logger.info("Nenhum novo resultado detectado")
                else:
                    logger.warning("Nenhum resultado extraído")
                
                # Salva os resultados
                scraper.save_results()
                
                # Reseta contador de tentativas em caso de sucesso
  
