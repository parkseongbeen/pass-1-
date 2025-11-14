# sic-assembler-project
// pass1.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// DFD의 Symbol table
FILE *symtab_file; 
// DFD의 Intermediate file
FILE *intermediate_file; 
FILE *input_file;

// P1_assign_loc을 위한 Location Counter
int LOCCTR; 
int start_address = 0;
int program_length = 0;

// DFD의 P1_read_source
void process_line(char* label, char* opcode, char* operand) {
    
    // 주석 처리 (';'로 시작하는 라인 무시)
    if (label[0] == ';') {
        fprintf(intermediate_file, "%s", label); // 주석은 그대로 중간파일에 쓴다
        return;
    }
    
    // 중간 파일에 현재 주소(LOCCTR) 쓰기
    fprintf(intermediate_file, "%04X\t%s\t%s\t%s\n", LOCCTR, label, opcode, operand);

    // DFD의 P1_assign_sym
    if (strcmp(label, "-") != 0) {
        // 심볼 테이블에 저장
        fprintf(symtab_file, "%s\t%04X\n", label, LOCCTR); 
    }

    // DFD의 P1_assign_loc (주소 할당)
    if (strcmp(opcode, "WORD") == 0) {
        LOCCTR += 3;
    } else if (strcmp(opcode, "RESW") == 0) {
        LOCCTR += (3 * atoi(operand));
    } else if (strcmp(opcode, "RESB") == 0) {
        LOCCTR += atoi(operand);
    } else if (strcmp(opcode, "BYTE") == 0) {
        if (operand[0] == 'X') {
            LOCCTR += (strlen(operand) - 3) / 2; // X'F1' -> 1바이트
        } else if (operand[0] == 'C') {
            LOCCTR += (strlen(operand) - 3); // C'EOF' -> 3바이트
        }
    } else {
        LOCCTR += 3; // 기본 명령어는 3바이트
    }
}

int main() {
    char line[256];
    char label[50], opcode[50], operand[50];

    // DFD의 Source Program
    if ((input_file = fopen("input.asm", "r")) == NULL) {
        printf("ERROR: input.asm 파일을 열 수 없습니다.\n");
        return 1;
    }
    
    intermediate_file = fopen("intermediate.txt", "w");
    symtab_file = fopen("symtab.txt", "w");

    printf("PASS-1 시작...\n");

    // 첫 줄 처리 (START)
    fgets(line, sizeof(line), input_file);
    sscanf(line, "%s\t%s\t%s", label, opcode, operand);

    if (strcmp(opcode, "START") == 0) {
        start_address = (int)strtol(operand, NULL, 16);
        LOCCTR = start_address;
    } else {
        LOCCTR = 0;
    }
    
    // DFD의 Access intermediate file
    fprintf(intermediate_file, "%04X\t%s\t%s\t%s\n", start_address, label, opcode, operand);

    // DFD의 P1_read_source (라인 단위 처리)
    while (fgets(line, sizeof(line), input_file) != NULL) {
        // 빈 줄이거나 탭만 있는 경우 처리
        if (sscanf(line, "%s\t%s\t%s", label, opcode, operand) < 2) {
             if (line[0] == ';') { // 주석 처리
                strcpy(label, line);
                process_line(label, "", "");
             }
             continue;
        }

        if (strcmp(opcode, "END") == 0) {
            fprintf(intermediate_file, "-\t%s\t%s\t%s\n", label, opcode, operand);
            break;
        }
        
        process_line(label, opcode, operand);
    }
    
    program_length = LOCCTR - start_address;
    printf("프로그램 길이: %d (0x%X)\n", program_length, program_length);

    fclose(input_file);
    fclose(intermediate_file);
    fclose(symtab_file);

    printf("PASS-1 완료. 'intermediate.txt'와 'symtab.txt' 생성됨.\n");
    return 0;
}
