#include <stdio.h>
#include <stdlib.h>

// Definição da estrutura para armazenar uma imagem PGM
typedef struct {
    int largura;        // Largura da imagem (número de pixels na horizontal)
    int altura;         // Altura da imagem (número de pixels na vertical)
    unsigned char *dados; // Vetor que armazena os dados da imagem (intensidade dos pixels)
} ImagemPGM;

// Função para ler uma imagem PGM de um arquivo
ImagemPGM* lerImagemPGM(const char *nomeArquivo) {
    FILE *arquivo = fopen(nomeArquivo, "rb"); // Abre o arquivo para leitura binária
    if (!arquivo) { // Se não conseguir abrir o arquivo, exibe erro
        perror("Erro ao abrir o arquivo");
        return NULL;
    }

    // Aloca memória para a estrutura da imagem
    ImagemPGM *imagem = (ImagemPGM *)malloc(sizeof(ImagemPGM));
    if (!imagem) { // Se não conseguir alocar memória, exibe erro
        perror("Erro ao alocar memória");
        fclose(arquivo); // Fecha o arquivo aberto
        return NULL;
    }

    char tipo[3];
    fscanf(arquivo, "%s", tipo); // Lê o tipo de arquivo PGM (P2 ou P5)
    
    if (tipo[0] == 'P' && tipo[1] == '5') { // Se for P5 (formato binário)
        fscanf(arquivo, "%d %d", &imagem->largura, &imagem->altura); // Lê a largura e altura
        int maxValor;
        fscanf(arquivo, "%d", &maxValor); // Lê o valor máximo de intensidade (geralmente 255)
        fgetc(arquivo); // Consome o caractere de nova linha

        // Aloca memória para os dados da imagem
        imagem->dados = (unsigned char *)malloc(imagem->largura * imagem->altura);
        fread(imagem->dados, 1, imagem->largura * imagem->altura, arquivo); // Lê os dados da imagem
    } else if (tipo[0] == 'P' && tipo[1] == '2') { // Se for P2 (formato ASCII)
        fscanf(arquivo, "%d %d", &imagem->largura, &imagem->altura); // Lê a largura e altura
        int maxValor;
        fscanf(arquivo, "%d", &maxValor); // Lê o valor máximo de intensidade
        fgetc(arquivo); // Consome o caractere de nova linha

        // Aloca memória para os dados da imagem
        imagem->dados = (unsigned char *)malloc(imagem->largura * imagem->altura);
        for (int i = 0; i < imagem->largura * imagem->altura; i++) { // Lê os valores dos pixels
            int valor;
            fscanf(arquivo, "%d", &valor); // Lê o valor do pixel
            imagem->dados[i] = (unsigned char)valor; // Armazena o valor do pixel
        }
    } else {
        fprintf(stderr, "Formato de arquivo não suportado\n"); // Se o tipo não for P2 ou P5
        fclose(arquivo);
        free(imagem); // Libera a memória alocada para a estrutura da imagem
        return NULL;
    }

    fclose(arquivo); // Fecha o arquivo
    return imagem; // Retorna a imagem lida
}

// Função para salvar uma imagem PGM em um arquivo
void salvarImagemPGM(const char *nomeArquivo, ImagemPGM *imagem) {
    FILE *arquivo = fopen(nomeArquivo, "wb"); // Abre o arquivo para escrita binária
    if (!arquivo) { // Se não conseguir abrir o arquivo, exibe erro
        perror("Erro ao abrir o arquivo para escrita");
        return;
    }

    // Escreve o cabeçalho do arquivo PGM (P5, largura, altura e valor máximo)
    fprintf(arquivo, "P5\n%d %d\n255\n", imagem->largura, imagem->altura);
    // Escreve os dados da imagem no arquivo
    fwrite(imagem->dados, 1, imagem->largura * imagem->altura, arquivo);
    fclose(arquivo); // Fecha o arquivo
}

// Função para calcular a mediana de um vetor de valores
unsigned char mediana(unsigned char *v, int n) {
    // Ordena o vetor de valores
    for (int i = 0; i < n - 1; i++) {
        for (int j = 0; j < n - i - 1; j++) {
            if (v[j] > v[j + 1]) {
                unsigned char temp = v[j];
                v[j] = v[j + 1];
                v[j + 1] = temp;
            }
        }
    }
    return v[n / 2]; // Retorna o valor central (mediana)
}

// Função para aplicar o filtro de mediana na imagem
void aplicarFiltroMediana(ImagemPGM *imagem) {
    // Aloca memória para a nova imagem filtrada
    unsigned char *novaImagem = (unsigned char *)malloc(imagem->largura * imagem->altura);
    if (!novaImagem) { // Se não conseguir alocar memória, exibe erro
        perror("Erro ao alocar memória para nova imagem");
        return;
    }

    // Aplica o filtro de mediana pixel por pixel, excluindo as bordas
    for (int y = 1; y < imagem->altura - 1; y++) {
        for (int x = 1; x < imagem->largura - 1; x++) {
            unsigned char janela[9];
            int k = 0;

            // Preenche a janela 3x3 com os valores dos pixels vizinhos
            for (int j = -1; j <= 1; j++) {
                for (int i = -1; i <= 1; i++) {
                    janela[k++] = imagem->dados[(y + j) * imagem->largura + (x + i)];
                }
            }

            // Aplica a mediana à janela e armazena o resultado na nova imagem
            novaImagem[y * imagem->largura + x] = mediana(janela, 9);
        }
    }

    // Substitui os dados da imagem original pelos dados da imagem filtrada
    free(imagem->dados);
    imagem->dados = novaImagem;
}

// Função para liberar a memória alocada para a imagem
void liberarImagem(ImagemPGM *imagem) {
    free(imagem->dados); // Libera a memória dos dados da imagem
    free(imagem); // Libera a memória da estrutura da imagem
}

// Função principal do programa
int main() {
    // Lê a imagem "cameraman.pgm"
    ImagemPGM *imagem = lerImagemPGM("cameraman.pgm");
    if (!imagem) { // Se não conseguir ler a imagem, termina o programa
        return 1;
    }

    // Aplica o filtro de mediana na imagem
    aplicarFiltroMediana(imagem);

    // Salva a imagem filtrada como "cameraman_mediana.pgm"
    salvarImagemPGM("cameraman_mediana.pgm", imagem);

    // Libera a memória da imagem
    liberarImagem(imagem);

    // Exibe uma mensagem indicando que o filtro foi aplicado e a imagem foi salva
    printf("Filtro de mediana aplicado e imagem salva como cameraman_mediana.pgm\n");
    return 0;
}
