```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define MASK_LSB 0xFE
#define NULL_BYTE "00000000"
#define BIN_LEN 1000
#define MESG_LEN 200

void binconv(char ch, char *bin) {
    for (int i = 7; i >= 0; i--) {
        bin[7 - i] = (ch & (1 << i)) ? '1' : '0';
    }
    bin[8] = '\0';
}
typedef struct {
    unsigned char *data;
    long size;
    int pixeldat;
} BMPFile;
BMPFile readfile(const char *filename) {
    BMPFile bmp = {NULL, 0, 0};
    FILE *file = fopen(filename, "rb");
    if (!file) {
        printf("Error opening file: %s\n", filename);
        return bmp;
    }
    fseek(file, 0, SEEK_END);
    bmp.size = ftell(file);
    rewind(file);
    bmp.data = (unsigned char *)malloc(bmp.size);
    if (bmp.data) {
        fread(bmp.data, 1, bmp.size, file);
        bmp.pixeldat = bmp.data[10] |
                       (bmp.data[11] << 8) |
                       (bmp.data[12] << 16) |
                       (bmp.data[13] << 24);
    }
    fclose(file);
    return bmp;
}void writefile(const char *filename, unsigned char *buffer, long size) {
    FILE *file = fopen(filename, "wb");
    if (!file) {
        printf("\nError writing to file: Cannot open file\n");
        return;
    }
    fwrite(buffer, 1, size, file);
    fclose(file);
}void encmesg(const char *inputFile, const char *outputFile, const char *mesg) {
    BMPFile bmp = readfile(inputFile);
    if (!bmp.data) return;

    char binmesg[BIN_LEN] = "";
    for (int i = 0; mesg[i] != '\0'; i++) {
        char bin[9];
        binconv(mesg[i], bin);
        strcat(binmesg, bin);
    }
    strcat(binmesg, NULL_BYTE);
    int bitind = 0;
    int totbit = strlen(binmesg);
    for (long i = bmp.pixeldat; i < bmp.size && bitind < totbit; i++) {
        bmp.data[i] = (bmp.data[i] & MASK_LSB) | (binmesg[bitind] - '0');
        bitind++;
    }
    writefile(outputFile, bmp.data, bmp.size);
    free(bmp.data);
}void decodmesg(const char *inputFile) {
    BMPFile bmp = readfile(inputFile);
    if (!bmp.data) return;
    char binmesg[BIN_LEN] = "";
    int bitind = 0;
    for (long i = bmp.pixeldat; i < bmp.size && bitind < BIN_LEN; i++, bitind++) {
        if ((bmp.data[i] & 1) == 1) {
            binmesg[bitind] = '1';
        } else {
            binmesg[bitind] = '0';
        }
    }
    binmesg[bitind] = '\0';
    free(bmp.data);
    char extractedMesg[MESG_LEN] = "";
    for (int i = 0; i < bitind; i += 8) {
        char charbin[9];
        strncpy(charbin, &binmesg[i], 8);
        charbin[8] = '\0';
        extractedMesg[i / 8] = (char)strtol(charbin, NULL, 2);
        if (extractedMesg[i / 8] == '\0') break;
    }

    printf("\nHidden message is : %s\n", extractedMesg);
}int main() {
    int ch;
    char mesg[MESG_LEN];

    printf("\nWelcome to the Steganography Tool");
    printf("\n1. Encode message");
    printf("\n2. Decode message");
    printf("\nEnter your choice: ");
    scanf("%d", &ch);
    getchar();

    if (ch == 1) {
        printf("Enter message: ");
        fgets(mesg, sizeof(mesg), stdin);
        mesg[strcspn(mesg, "\n")] = '\0';
        encmesg("input.bmp", "output.bmp", mesg);
        printf("\nMessage sucessfuly encoded");
    } else if (ch == 2) {
        decodmesg("output.bmp");
    } else {
        printf("\nInvalid choice");
    }
    return 0;
}



