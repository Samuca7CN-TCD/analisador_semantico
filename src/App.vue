<script setup>
import { ref } from 'vue'
import Button from './components/Button.vue'

const program = ref("") // Código hipotético
const stack = ref([]) // Pilha de escopos
const responses = ref([]) // Respostas de erro e printes de variáveis

const analisador_semantico = () => {
  // Dividir código em linhas
  let program_aux = program.value.split(/\s*\n\s*/)

  //Reiniciar variáveis
  stack.value = []
  responses.value = []

  program_aux.forEach((linha, index) => { // Para cada linha...

    //if (!linha.length) return

    //Reiniciar variáveis
    let tokens = linha.replaceAll(',', ' , ').split(/[\s=]+/)
    let top = null
    let flag = 0
    let attr_type = ""

    // Se houver qualquer código fora de um escopo: ERRO!
    if (tokens[0] != 'BLOCO' && !stack.value.length) {
      push_responses(0, index)
      return
    }

    switch (tokens[0]) {

      case 'BLOCO': // Criar novo escopo
        stack.value.push({
          'iden': tokens[1],
          'vars': {}
        })
        break;

      case 'NUMERO':
      case 'CADEIA': // Declaração de variáveis
        top = stack.value.pop(); // Pega escopo mais interno da pilha

        for (let i = 1; i < tokens.length; i += 2) { // Para cada para de valores...

          if (tokens[i] == ',') { // Se a atribuição cair na virgula, decrementar i para na próxima atribuição pegarmos a 2ª variável
            i -= 1
            continue
          }

          if (top.vars[tokens[i]]) { // Se a variável já existe no escopo, erro!
            push_responses(1, index, tokens[i])
            continue
          }

          if (tokens[i + 1] && tokens[i + 1] != ',') { // Se há atribuição...

            // INICIO verificar se o valor da atribuição é correspondente ao tipo de variável
            if (tokens[0] == 'NUMERO' && !/^[+\-]?\d+(\.\d+)?$/.test(tokens[i + 1])) {
              push_responses(2, index)
              continue
            } else if (tokens[0] == 'CADEIA' && !/^“(.*)”$|^"(.*)"$/u.test(tokens[i + 1])) {
              push_responses(3, index)
              continue
            }
            // FIM verificar se o valor da atribuição é correspondente ao tipo de variável


            top.vars[tokens[i]] = { // Guardar variável COM atribuição, no escopo
              'valor': tokens[i + 1],
              'tipo': tokens[0]
            }

          } else { // Se NÃO há atribuição...
            top.vars[tokens[i]] = { // Guardar variável SEM atribuição, no escopo
              'valor': '',
              'tipo': tokens[0]
            }
          }
        }
        stack.value.push(top); // Devolver escopo atualizado (ou não) à pilha
        break;

      case 'PRINT': // Printar valor da variável
        flag = 0 // Determina se a variável foi ou não encontrada em algum escopo

        for (var escopo of stack.value.slice().reverse()) { // para cada escopo aberto (interno -> externo)...
          if (!flag && escopo.vars[tokens[1]]) { // se encontrar a variável, printar seu valor e parar de provurar nos escopos
            push_responses(4, index, tokens[1], escopo)
            flag = 1
            break
          }
        }

        // Se não encontrar a variável em nenum escopo: ERRO!
        if (!flag) responses.value.push("Erro na linha " + (index + 1) + ": Variável '" + tokens[1] + "' não foi declarada!")
        break;

      case 'FIM': // Fechamento de ESCOPO
        top = stack.value.pop() // Pegar escopo mais interno
        if (tokens[1] != top.iden) { // Se o identificador do escpopo mais interno não foi o identificador de fechamento atual: ERRO!
          push_responses(5, index, tokens[1], null, top)
          stack.value.push(top) // Devolve escopo à pilha
        }
        break;

      default: // Atribuição de variável já declarada
        flag = 0 // Determina se a variável foi ou não encontrada em algum escopo

        for (var escopo of stack.value.slice().reverse()) { // Procurar variável nos escopos (interno -> externo)
          if (escopo.vars[tokens[0]]) { // Achou a variavel no escopo
            if (/^[a-zA-Z]$/.test(tokens[1])) { // O valor atribuido é uma outra variável?
              for (var escopo_2 of stack.value.slice().reverse()) { // Procurar variável2 nos escopos (interno -> externo)
                if (escopo_2.vars[tokens[1]]) { // Achou a variável2 no escopo
                  if (escopo.vars[tokens[0]]['tipo'] == escopo_2.vars[tokens[1]]['tipo']) { // se ambas as variáveis forem do mesmo tipo:
                    escopo.vars[tokens[0]]['valor'] = escopo_2.vars[tokens[1]]['valor'] // O valor da variável1 recebe o valor da variável2
                  } else {
                    push_responses(6, index)
                  }
                  flag = 1 // marca variavel2 como achada
                  break // sair da procura de escopos da variável2
                }
              }

              // Se não achou a variável2: ERRO!
              if (!flag) push_responses(7, index, tokens[1])

            } else { // Se a atribuição for um dado, receber esse dado na variável

              if (/^[+\-]?\d+(\.\d+)?$/.test(tokens[1])) attr_type = "NUMERO" // seta o tipo da atribuição como NUMERO
              else if (/^[+\-]?\d+(\.\d+)?$/.test(tokens[1])) attr_type = "CADEIA" // Seta o tipo da atribuição como CADEIA

              if (escopo.vars[tokens[0]].tipo != attr_type) push_responses(6, index) // verifica se a tribuição édo mesmo tipo que a variável
              else escopo.vars[tokens[0]].valor = tokens[1] // faz a atribuição
            }
            flag = 1 // marca variavel1 como achada
            break // sair da procura de escopos da variável1
          }
        }
        // Se não achou a variável1: ERRO!
        if (!flag) push_responses(7, index, tokens[0])
    }
  })

  // Caso ainda haja escopos na pilha após finalização da análise:
  if (stack.value.length) push_responses(8)

}

const push_responses = (id, index = null, token = null, escopo = null, top = null) => {
  switch (id) {
    case 0: responses.value.push("Erro na linha " + (index + 1) + ": Linha fora de escopo!"); break
    case 1: responses.value.push("Erro na linha " + (index + 1) + ": Variável '" + token + "' já foi declarada!"); break
    case 2: responses.value.push("Erro na linha " + (index + 1) + ": O valor atribuído NÃO é um NÚMERO INTEIRO ou REAL"); break
    case 3: responses.value.push("Erro na linha " + (index + 1) + ": O valor atribuído NÃO é uma SEQUÊNCIA DE CARACTERES entre ASPAS DUPLAS"); break
    case 4: responses.value.push("Linha " + (index + 1) + " (printar valor de " + token + "): " + escopo.vars[token]['valor']); break
    case 5: responses.value.push("Erro na linha " + (index + 1) + ": O FIM " + token + " não é o fechamento do escopo do BLOCO " + top.iden); break
    case 6: responses.value.push("Erro na linha " + (index + 1) + ": A variável não é do mesmo tipo que o valor atribuído"); break
    case 7: responses.value.push("Erro na linha " + (index + 1) + ": Variável '" + token + "' não foi declarada!"); break
    case 9: responses.value.push("Erro: Há escopos que não foram fechados!"); break
  }
}

</script>

<template>
  <div class="w-full text-center font-bold my-5">
    <h1 class="text-5xl">Trabalho de Compiladores</h1>
    <h2 class="text-2xl">Samuel Correia Nascimento</h2>
    <h2 class="text-xl">201920081</h2>
  </div>
  <div class="w-full flex flex-row align-middle p-10 space-x-10">
    <div class="w-full" :class="{ 'md:w-1/2': stack.length }">
      <form method="post" @submit.prevent="analisador_semantico">
        <Button class="mb-2">Analisar</Button>
        <textarea v-model="program" class="w-full border-2 rounded-xl p-5" rows="40"
          placeholder="Insira um programa"></textarea>
        <Button>Analisar</Button>
      </form>
    </div>
    <div v-if="responses.length" class="w-full md:w-1/2 border-2 rounded-xl p-3">
      <ul class="divide-y-2">
        <li v-for="response in responses" class="py-5" :class="{ 'text-red-500': response[0] == 'E' }">
          {{ response }}
        </li>
      </ul>
    </div>
  </div>
  Pilha final:
  <pre>{{ stack }}</pre>
</template>
