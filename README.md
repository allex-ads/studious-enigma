import tkinter as tk
import random
import math
import ast
import operator
import re
import threading

# Som: winsound só existe no Windows. Nos outros sistemas usamos o "bell" da janela.
try:
    import winsound
    SOM_ATIVO = True
except ImportError:
    SOM_ATIVO = False

PERMITIDOS = set("0123456789.+-*/() ")
MAX_EXPRESSAO = 100

# Indica se o visor está mostrando algo que não é uma expressão (erro, BIN, sorte)
visor_especial = False


# ---------------------------------------------------------------
# AVALIADOR SEGURO (substitui o eval)
# ---------------------------------------------------------------
OPERADORES_BIN = {
    ast.Add: operator.add,
    ast.Sub: operator.sub,
    ast.Mult: operator.mul,
    ast.Div: operator.truediv,
    ast.Pow: operator.pow,
}
OPERADORES_UN = {
    ast.UAdd: operator.pos,
    ast.USub: operator.neg,
}


def _avaliar_no(no):
    if isinstance(no, ast.Expression):
        return _avaliar_no(no.body)
    if isinstance(no, ast.Constant) and type(no.value) in (int, float):
        return no.value
    if isinstance(no, ast.BinOp) and type(no.op) in OPERADORES_BIN:
        esq = _avaliar_no(no.left)
        dir_ = _avaliar_no(no.right)
        if isinstance(no.op, ast.Pow) and (abs(dir_) > 1000 or abs(esq) > 1e6):
            raise ValueError("potência grande demais")
        resultado = OPERADORES_BIN[type(no.op)](esq, dir_)
        if isinstance(resultado, complex):  # ex.: (-8)**0.5 vira número complexo
            raise ValueError("resultado complexo")
        return resultado
    if isinstance(no, ast.UnaryOp) and type(no.op) in OPERADORES_UN:
        return OPERADORES_UN[type(no.op)](_avaliar_no(no.operand))
    raise ValueError("expressão não permitida")


def avaliar(expressao):
    """Calcula uma expressão matemática simples sem usar eval()."""
    expressao = expressao.strip().replace(",", ".")
    if not expressao or len(expressao) > MAX_EXPRESSAO:
        raise ValueError("expressão inválida")
    # Remove zeros à esquerda (ex.: 007 -> 7), que o Python rejeitaria
    expressao = re.sub(r"(?<![\d.])0+(?=\d)", "", expressao)
    arvore = ast.parse(expressao, mode="eval")
    return _avaliar_no(arvore)


def formatar(numero):
    """Deixa o resultado bonito: sem '.0' e sem erros de ponto flutuante."""
    if isinstance(numero, float):
        if math.isnan(numero) or math.isinf(numero):
            raise ValueError("resultado inválido")
        texto = f"{numero:.12g}"  # corta o ruído (0.30000000000000004 -> 0.3)
        valor = float(texto)
        if valor.is_integer() and abs(valor) < 1e15:
            return str(int(valor))
        return texto
    return str(numero)


# ---------------------------------------------------------------
# SOM (não trava a interface)
# ---------------------------------------------------------------
def _beep(freq, duracao):
    try:
        winsound.Beep(freq, duracao)
    except Exception:
        pass


def tocar_som(freq=600, duracao=40, importante=False):
    if SOM_ATIVO:
        threading.Thread(target=_beep, args=(freq, duracao), daemon=True).start()
    elif importante:
        try:
            janela.bell()
        except Exception:
            pass


# ---------------------------------------------------------------
# FUNÇÕES DO VISOR
# ---------------------------------------------------------------
def set_visor(texto, especial=False):
    global visor_especial
    visor.delete(0, tk.END)
    visor.insert(0, texto)
    visor_especial = especial


def limpar_se_especial():
    global visor_especial
    if visor_especial:
        visor.delete(0, tk.END)
        visor_especial = False


def status(texto, cor):
    status_label.config(text=texto, fg=cor)


def adicionar_texto(char):
    tocar_som(700, 30)
    limpar_se_especial()
    visor.insert(tk.END, str(char))
    visor.focus_set()


def apagar():
    tocar_som(500, 30)
    if visor_especial:
        set_visor("")
    else:
        atual = visor.get()
        visor.delete(0, tk.END)
        visor.insert(0, atual[:-1])
    visor.focus_set()


def limpar(event=None):
    tocar_som(300, 60)
    set_visor("")
    status("SISTEMA PRONTO // MODO NEON", "#00ffcc")
    visor.focus_set()


def erro(mensagem="FALHA CRÍTICA NA EXPRESSÃO"):
    tocar_som(200, 200, importante=True)
    set_visor("ERRO DE SISTEMA", especial=True)
    status(mensagem, "#ff0055")


def calcular(event=None):
    if visor_especial:
        return
    try:
        resultado = avaliar(visor.get())
        texto = formatar(resultado)
    except ZeroDivisionError:
        erro("DIVISÃO POR ZERO DETECTADA")
        return
    except Exception:
        erro()
        return
    tocar_som(1000, 80, importante=True)
    set_visor(texto)
    status("CÁLCULO CONCLUÍDO COM SUCESSO", "#39ff14")


def raiz_quadrada():
    if visor_especial:
        return
    try:
        valor = avaliar(visor.get())
        if valor < 0:
            erro("RAIZ DE NÚMERO NEGATIVO")
            return
        resultado = formatar(math.sqrt(valor))
    except Exception:
        erro()
        return
    tocar_som(950, 60)
    set_visor(resultado)
    status("RAIZ QUADRADA CALCULADA", "#39ff14")


def porcentagem():
    if visor_especial:
        return
    try:
        resultado = formatar(avaliar(visor.get()) / 100)
    except Exception:
        erro()
        return
    tocar_som(950, 60)
    set_visor(resultado)
    status("PORCENTAGEM APLICADA (÷100)", "#39ff14")


# Funções que não existem em calculadoras comuns
def converter_binario():
    try:
        valor = avaliar(visor.get())
        num = int(valor)
        sinal = "-" if num < 0 else ""
        texto = f"BIN: {sinal}{bin(abs(num))[2:]}"
    except Exception:
        tocar_som(200, 120, importante=True)
        status("DIGITE UM NÚMERO INTEIRO ANTES", "#ff0055")
        return
    tocar_som(900, 50)
    set_visor(texto, especial=True)
    status("TRADUZIDO PARA LINGUAGEM DE MÁQUINA", "#ff00ff")


def rolar_dado_d20():
    tocar_som(850, 40)
    valor = random.randint(1, 20)
    set_visor(str(valor))
    if valor == 20:
        status("ACERTO CRÍTICO! (D20 = 20)", "#39ff14")
    elif valor == 1:
        status("FALHA CRÍTICA! (D20 = 1)", "#ff0055")
    else:
        status(f"DADO D20 ROLADO: {valor}", "#00ffcc")


def modo_oraculo():
    tocar_som(1200, 100)
    sorte = random.randint(0, 100)
    set_visor(f"SORTE: {sorte}%", especial=True)
    status("PROBABILIDADE DE SUCESSO HOJE", "#ffff00")


# ---------------------------------------------------------------
# TECLADO
# ---------------------------------------------------------------
def filtrar_tecla(event):
    """Só deixa digitar números e operadores; Enter/Esc/Backspace funcionam."""
    # Atalhos com Ctrl (copiar/colar etc.) seguem o comportamento normal
    if event.state & 0x4:
        return None

    if event.keysym == "BackSpace":
        if visor_especial:
            set_visor("")
            return "break"
        return None

    if event.keysym in ("Return", "KP_Enter", "Escape"):
        return None  # tratados pelos binds da janela

    char = event.char
    if not char or ord(char) < 32:
        return None  # setas, Home, End, Tab, etc.

    if char == ",":
        char = "."

    if char in PERMITIDOS:
        limpar_se_especial()
        if char != event.char:  # vírgula virou ponto
            visor.insert(tk.INSERT, char)
            return "break"
        return None

    return "break"  # bloqueia qualquer outro caractere


# ---------------------------------------------------------------
# CONFIGURAÇÃO DA JANELA
# ---------------------------------------------------------------
janela = tk.Tk()
janela.title("CYBER-CALC 3000")
janela.geometry("360x600")
janela.configure(bg="#0d0221")  # Roxo ultra escuro
janela.resizable(False, False)

status_label = tk.Label(
    janela,
    text="SISTEMA PRONTO // MODO NEON",
    font=("Consolas", 9, "bold"),
    bg="#0d0221",
    fg="#00ffcc",
)
status_label.pack(pady=(12, 0))

visor = tk.Entry(
    janela,
    font=("Consolas", 22, "bold"),
    bg="#19053a",
    fg="#39ff14",
    insertbackground="#39ff14",
    borderwidth=3,
    relief="solid",
    justify="right",
)
visor.pack(padx=15, pady=15, fill="x", ipady=12)
visor.bind("<Key>", filtrar_tecla)
visor.focus_set()

janela.bind("<Return>", calcular)
janela.bind("<KP_Enter>", calcular)
janela.bind("<Escape>", limpar)

frame_botoes = tk.Frame(janela, bg="#0d0221")
frame_botoes.pack(padx=15, pady=(0, 15), fill="both", expand=True)


def criar_botao(texto, comando, cor, linha, coluna, fonte_tam=15,
                bg="#14042e", colspan=1, fg=None, active_bg=None):
    btn = tk.Button(
        frame_botoes,
        text=texto,
        font=("Consolas", fonte_tam, "bold"),
        bg=bg,
        fg=fg or cor,
        activebackground=active_bg or cor,
        activeforeground="#0d0221",
        command=comando,
        relief="ridge",
        borderwidth=2,
        takefocus=0,
    )
    btn.grid(row=linha, column=coluna, columnspan=colspan,
             sticky="nsew", padx=3, pady=3, ipady=8)
    return btn


# Linha 0: botões especiais
especiais = [
    ("BIN", converter_binario, "#ff00ff"),
    ("D20", rolar_dado_d20, "#00ffcc"),
    ("ORÁCULO", modo_oraculo, "#ffff00"),
    ("C", limpar, "#ff0055"),
]
for col, (texto, func, cor) in enumerate(especiais):
    criar_botao(texto, func, cor, 0, col, fonte_tam=10, bg="#1a0836")

# Linha 1: parênteses, backspace e raiz
criar_botao("(", lambda: adicionar_texto("("), "#ff00ff", 1, 0)
criar_botao(")", lambda: adicionar_texto(")"), "#ff00ff", 1, 1)
criar_botao("⌫", apagar, "#ff0055", 1, 2)
criar_botao("√", raiz_quadrada, "#ffff00", 1, 3)

# Números e operadores
botoes = [
    ('7', 2, 0, "#00ffcc"), ('8', 2, 1, "#00ffcc"), ('9', 2, 2, "#00ffcc"), ('/', 2, 3, "#ff00ff"),
    ('4', 3, 0, "#00ffcc"), ('5', 3, 1, "#00ffcc"), ('6', 3, 2, "#00ffcc"), ('*', 3, 3, "#ff00ff"),
    ('1', 4, 0, "#00ffcc"), ('2', 4, 1, "#00ffcc"), ('3', 4, 2, "#00ffcc"), ('-', 4, 3, "#ff00ff"),
    ('0', 5, 0, "#00ffcc"), ('.', 5, 1, "#00ffcc"), ('+', 5, 3, "#ff00ff"),
]
for texto, linha, coluna, cor in botoes:
    criar_botao(texto, lambda t=texto: adicionar_texto(t), cor, linha, coluna)

# Porcentagem
criar_botao("%", porcentagem, "#ffff00", 5, 2)

# Igual neon (linha inteira)
criar_botao("=", calcular, "#39ff14", 6, 0, fonte_tam=16, bg="#39ff14",
            colspan=4, fg="#0d0221", active_bg="#00ffcc")

for i in range(7):
    frame_botoes.rowconfigure(i, weight=1)
for i in range(4):
    frame_botoes.columnconfigure(i, weight=1)

janela.mainloop()
