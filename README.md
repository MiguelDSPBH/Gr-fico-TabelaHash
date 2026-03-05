import hashlib
import multiprocessing
import time
import matplotlib.pyplot as plt

def worker_bench(inicio, fim):
    """Função de processamento puro (CPU-bound) para o benchmark."""
    for i in range(inicio, fim):
        senha_teste = f"{i:09d}"
        # Apenas gera o hash para consumir ciclos de CPU
        hashlib.md5(senha_teste.encode()).hexdigest()

def rodar_teste(n_processos, limite):
    if n_processos == 1:
        # Execução Serial Pura
        inicio_t = time.time()
        worker_bench(0, limite)
        return time.time() - inicio_t
    else:
        # Execução Paralela usando Multiprocessing
        intervalo = limite // n_processos
        processos = []
        inicio_t = time.time()
        
        for i in range(n_processos):
            # Garante que o último processo pegue o resto da divisão se houver
            fim = (i + 1) * intervalo if i != n_processos - 1 else limite
            p = multiprocessing.Process(target=worker_bench, args=(i * intervalo, fim))
            processos.append(p)
            p.start()
            
        for p in processos:
            p.join()
        return time.time() - inicio_t

if __name__ == "__main__":
    # Configurações do Teste
    limite_benchmark = 5_000_000 # Aumente para 10M se tiver um PC muito rápido
    # Testaremos de 1 até o máximo de cores lógicos do seu PC
    max_cores = multiprocessing.cpu_count()
    threads_testar = [1, 2, 4, 8] 
    if max_cores > 8: threads_testar.append(max_cores)
    
    tempos = []

    print(f"--- Iniciando Benchmark ({limite_benchmark} hashes) ---")
    print(f"Detectados {max_cores} núcleos no sistema.\n")
    
    for n in threads_testar:
        print(f"Testando com {n} processo(s)...", end=" ", flush=True)
        t = rodar_teste(n, limite_benchmark)
        tempos.append(t)
        print(f"{t:.4f}s")

    # --- Cálculos de Performance ---
    t_serial = tempos[0]
    speedups = [t_serial / tn for tn in tempos]
    eficiencias = [(s / n) * 100 for s, n in zip(speedups, threads_testar)]

    # --- Geração dos Gráficos ---
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(14, 6))
    plt.subplots_adjust(wspace=0.3)

    # 1. Gráfico de Speedup (Lei de Amdahl)
    ax1.plot(threads_testar, speedups, marker='o', color='royalblue', linewidth=2, label='Speedup Real')
    ax1.plot(threads_testar, threads_testar, color='tomato', linestyle='--', label='Speedup Ideal (Linear)')
    ax1.set_title('Gráfico de Speedup', fontsize=14, pad=15)
    ax1.set_xlabel('Número de Processos (Cores)')
    ax1.set_ylabel('Speedup ($T_1 / T_n$)')
    ax1.legend()
    ax1.grid(True, alpha=0.3)

    # 2. Gráfico de Eficiência
    cores_barras = ['#4CAF50' if e > 70 else '#FFC107' if e > 40 else '#F44336' for e in eficiencias]
    bars = ax2.bar([str(n) for n in threads_testar], eficiencias, color=cores_barras, alpha=0.8)
    ax2.set_title('Eficiência do Paralelismo', fontsize=14, pad=15)
    ax2.set_xlabel('Número de Processos (Cores)')
    ax2.set_ylabel('Eficiência (%)')
    ax2.set_ylim(0, 115)

    # Adiciona os rótulos de porcentagem nas barras
    for bar in bars:
        height = bar.get_height()
        ax2.text(bar.get_x() + bar.get_width()/2., height + 2,
                f'{height:.1f}%', ha='center', va='bottom', fontweight='bold')

    print("\n[INFO] Teste finalizado. Feche a janela do gráfico para encerrar o programa.")
    plt.show()
    
