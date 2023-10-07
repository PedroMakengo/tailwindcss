<script setup lang="ts">
import { onMounted, ref, watch, watchEffect } from "vue";
import { useField, useForm } from "vee-validate";

import { UsecaseObterTipoPagamento } from "@/Domain/Usecases/isRental/usecase_lista_tipopagamento";
import { UsecaseObterVendedores } from "@/Domain/Usecases/isRental/usecase_lista_vendedores";
import { FormatarDataEditavel } from "@/utils/formatarDataEditavel";
import { UsecaseObterTiposDocumentos } from "@/Domain/Usecases/isRental/usecase_lista_tipo_documentos";

import { ITClientes } from "@/Domain/Entities/isRental/Clientes";

import { UsecaseObterClientesPaginacao } from "@/Domain/Usecases/isRental/Usecase_lista_cliente_paginacao";
import { ObterDataAtual } from "@/utils/obterDataAtual";
import { FormatarData } from "@/utils/formatarData";
import { FormatarPrecoSemPontoVirgula } from "@/utils/formatarPrecoSemPontoVirgula";
import { compararDuasData } from "@/utils/compararDuasData";

import { FormatarPreco } from "@/utils/formatarPreco";

import { useToast } from "vue-toastification";
import { UsecaseObterListaArtigosEmUrso } from "@/Domain/Usecases/Geral/usecase_lista_artigo_em_uso";
import { UsecaseBuscarCentroCusto } from "@/Domain/Usecases/isRental/usecase_lista_centro_custo";
import { DiferencaData } from "@/utils/diferencaEntreDatas";
import { contarDiasFacturadosEntreDatas } from "@/utils/contarDiasFaturados";
import { UsecaseObterFamilias } from "@/Domain/Usecases/isRental/usecase_lista_familias";
import { UsecaseObterArtigosFamilias } from "@/Domain/Usecases/isRental/usecase_lista_artigos_familia";

import { useStoreProforma } from "@/stores/Comercial/store_guardar_proforma";
import { useStoreProposta } from "@/stores/Comercial/store_proposta";

const storeProforma = useStoreProforma();

const storeProposta = useStoreProposta();

import TitleModal from "@/components/TitleModal/index.vue";

const toast = useToast();

interface Item {
  id: number;
  codArt: string;
  descricao: string;
  familia: string;
  tipo: string;
  iva: string;
  unidade: string;
  movStock: string;
  unidadeCompra: string;
  unidadeEntrada: string;
  unidadeSaida: string;
  unidadeVenda: string;
  integrado: null | any; // Aqui você pode ajustar o tipo de 'integrado' conforme necessário
  status: string;
  artigoAnulado: boolean;
}

interface ILinhaProposta {
  nlin: number;
  artigo: string;
  descricao: string;
  quantidade: number;
  qtdFacturada: number;
  precoUnit: number | string;
  codIva: number;
  taxa: number;
  desconto: number;
  total: number | string;
  dataInicio: string;
  dataFim: string;
}

interface ICentroCusto {
  id: string;
  centro: string;
  descricao: string;
  ano: string | number;
  inactivo: boolean;
  dataRegisto: string | null;
}

const { handleSubmit, resetForm } = useForm({
  validationSchema: {
    tipoDoc(value: String) {
      if (value?.length >= 1) return true;

      return "Campo é obrigatório.";
    },
    entidade(value: String) {
      if (value?.length >= 1) return true;

      return "Campo é obrigatório.";
    },
    condPag(value: String) {
      if (value?.length >= 2) return true;
      return "Campo é obrigatório.";
    },

    vendedor(value: String) {
      if (value?.length >= 2) return true;
      return "Campo é obrigatório.";
    },

    aoCuidado(value: any) {
      if (value?.length >= 2) return true;
      return "Campo é obrigatório.";
    },
    localAluguer(value: String) {
      if (value?.length >= 2) return true;
      return "Campo é obrigatório.";
    },
    telefone(value: any) {
      if (value?.length >= 2 && /[0-9-]+/.test(value)) return true;
      return "Campo é obrigatório.";
    },
    email(value: String) {
      if (value?.length >= 2) return true;
      return "Campo é obrigatório.";
    },
  },
});

const tipoDoc: any = useField("tipoDoc");
const client: any = useField<String | Number>("entidade");
const obs = ref("");
const condPagamento: any = useField("condPag");
const vendedor: any = useField("vendedor");
const aoCuidado: any = useField("aoCuidado");
const email: any = useField("email");
const telefone: any = useField("telefone");
const local: any = useField("localAluguer");
const mes: any = useField<boolean>("mes");
const norm = useField<boolean>("norm");
const qtdMeses = ref<number>(0);
const longa = useField<boolean>("longa");

const sabados = useField<boolean>("sabado");
const domingos = useField<boolean>("domingo");
const feriados = useField<boolean>("feriados");

const tiposPagamentos = ref([]);
const vendedores = ref([]);
const tipoDocumento = ref([]);
const aluguerItem = ref(["Normal", "Mensal", "Longa Duração"]);
const controlarAluguer = ref("Normal");
const tipoCliente = ref("");

const utilizador = ref("");

const aluguer = ref(false);
const ano = ObterDataAtual();

const conversaoProforma = ref(false);

const dataDocumento = ref(FormatarData(ano));

const handleChange = (item: String) => {
  item !== "Normal" ? (aluguer.value = true) : (aluguer.value = false);

  item === "Normal" ? (norm.value.value = true) : (norm.value.value = false);
  item === "Mensal" ? (mes.value.value = true) : (mes.value.value = false);
  item === "Longa Duração"
    ? (longa.value.value = true)
    : (longa.value.value = false);
};

const manterApenasPropriedadesDesejadas = (item: any) => {
  const propriedadesDesejadas = [
    "artigo",
    "codIva",
    "dataFim",
    "dataInicio",
    "desconto",
    "total",
    "precoUnit",
    "quantidade",
    "qtdFacturada",
    "taxa",
  ];

  const novoItem: any = {};
  propriedadesDesejadas.forEach((propriedade: string) => {
    if (item.hasOwnProperty(propriedade)) {
      console.log(propriedade);
      if (propriedade === "qtdFacturadaurada" || propriedade === "nlin") {
        novoItem[propriedade] = item[propriedade]
          ? Number(item[propriedade])
          : 0;
      } else {
        novoItem[propriedade] = item[propriedade];
      }
    }
  });

  return novoItem;
};

const addFormPedidoProposta = handleSubmit(async (values) => {
  if (conversaoProforma.value === true) {
    linhaTabela.value.forEach((item: any) => {
      item["total"] = Number(item["total"]);
      item["precoUnit"] = Number(item["precoUnit"]);
      item["qtdFacturadaurada"] = Number(item["qtdFacturada"]);
    });

    linhas.value = linhaTabela.value;
  } else {
    linhas.value.forEach((item: any) => {
      if (typeof item["precoUnit"] === "string") {
        item["precoUnit"] = FormatarPrecoSemPontoVirgula(item["precoUnit"]);
      }
    });
    linhas.value = linhas.value.map(manterApenasPropriedadesDesejadas);
  }
  const data = {
    ...values,
    qtdMeses: qtdMeses.value,
    obs: obs.value,
    tipoCliente: "",
    utilizador: utilizador.value,
    linhas: linhas.value,
  };

  if (linhas.value.length !== 0) {
    const response = await storeProposta.criarProposta(data);

    console.log(response);
    console.log("dados a mandar", data);

    if (response?.statusCode === 200) {
      if (response?.object.tipoDoc === "PRR")
        toast.success("Proposta adicionada com sucesso!", {
          timeout: 3000,
        });
      else {
        toast.success("Proforma adicionada com sucesso!", {
          timeout: 3000,
        });
      }
      familias.value = "";
      dataInicio.value = "";
      dataFim.value = "";
      qtdFacturada.value = 0;
      precoUnit.value = "0";
      desconto.value = 0;
      codIva.value = "";
      linhas.value = [];
      linhaTabela.value = [];
      centroCusto.value = "";
      resetForm();
    } else {
      toast.error("Não foi possível efetuar essa operação!", {
        timeout: 3000,
      });
    }
  } else {
    toast.warning("Não podes criar um documento sem adicionares linhas", {
      timeout: 3000,
    });
  }
});

// Pegar dados de Faturar
const clients = ref<ITClientes[]>([]);

const page = ref(1);
const take = ref(50);

const onIntersect = () => {
  page.value += 1;
  obterClientes();
};

const onIntersectCentroCusto = () => {
  page.value += 1;
  buscarCentroCusto();
};

async function obterClientes() {
  var response: ITClientes[] = await UsecaseObterClientesPaginacao.handler(
    `?page=${page.value}&take=${take.value}`
  );
  clients.value = [...clients.value, ...response];
  clients.value.forEach((item: any) => {
    item["nome_codigo"] = `${item["entidade"]} - ${item["nomeFiscal"]}`;
  });
}

async function obterTipoPagamentos() {
  const response = await UsecaseObterTipoPagamento.handler();
  tiposPagamentos.value = response;
}

async function obterVendedores() {
  const response = await UsecaseObterVendedores.handler();
  vendedores.value = response;
}

async function obterTipoDocumentos() {
  const response = await UsecaseObterTiposDocumentos.handler();
  tipoDocumento.value = response;
}

const obterNomeDoUtilizador = async () => {
  const buscandoUtilizador =
    (await localStorage.getItem("utilizador")) || "null";
  const user = JSON.parse(buscandoUtilizador);

  utilizador.value = user.nome;
};

function limparFormulario() {
  resetForm();
  limparCache();
  qtdMeses.value = 0;
  obs.value = "";
  linhaTabela.value = [];
}

// Migrar toda estrutura
//  Componentes da Tabela
const qtdMesesAluguer = ref(0);
const linhas = ref<ILinhaProposta[]>([]);
const linhaTabela = ref<ILinhaProposta[]>([]);
const dialog = ref(false);
const search = ref("");
const headers = ref([
  { key: "artigo", title: "Artigo" },
  { key: "descricao", title: "Descrição" },
  { key: "dataInicio", title: "Data Início" },
  { key: "dataFim", title: "Data Fim" },
  { key: "qtdFacturada", title: "Qtd Fact" },
  { key: "quantidade", title: "Qtd Alg" },
  { key: "precoUnit", title: "Preço Unit" },
  { key: "codIva", title: "IVA" },
  { key: "desconto", title: "Desconto" },
  { key: "total", title: "Total" },
  { key: "accoes", title: "Acções", sortable: false },
]);

var familias: any = ref("");
const artigo = ref<string>("");
const dataInicio = ref("");
const dataFim = ref("");
const qtdFacturada = ref(0);
const qtdAlg = ref(0);
const precoUnit = ref<string | number>("0");
const desconto = ref(0);
const codIva: any = ref<string>("");
const LinhasArtigo = ref<any>([]);
const titleOcupacao = ref("");
const dialogInformacaoArtigo = ref(false);
const listaFamilias = ref<any>([]);
const listaFamiliaArtigos = ref<Item[]>([]);
const descricao = ref("");
const total = ref<string | number>("0");
const centroCusto = ref("");
const listaCentroCusto = ref<ICentroCusto[]>([]);
const isVerificaDataInicio = ref(false);

const addLinhaPedidoProposta = () => {
  const totalFormatado = FormatarPrecoSemPontoVirgula(`${total.value}`);
  const precoFormatado = FormatarPrecoSemPontoVirgula(`${precoUnit.value}`);

  const dataLinha: any = {
    artigo: artigo.value,
    codIva: Number(codIva.value),
    desconto: Number(desconto.value),
    precoUnit: Number(precoFormatado),
    taxa: 0,
    total: Number(totalFormatado),
    qtdFacturada: Number(qtdFacturada.value),
    quantidade: Number(qtdAlg.value),
    nlin: 0,
    dataInicio: dataInicio.value,
    dataFim: dataFim.value,
  };

  // Linha Tabela
  const resDataComparada = compararDuasData(dataInicio.value, dataFim.value);

  if (resDataComparada) {
    const response = linhas.value.push(dataLinha);

    const dataTabela: any = {
      ...dataLinha,
      familia: familias.value,
      descricao: descricao.value,
      total: total.value,
    };

    linhaTabela.value.push(dataTabela);

    linhaTabela.value.forEach((item: any) => {
      item["precoUnit"] = FormatarPreco(item["precoUnit"]);
    });

    if (response !== 0) {
      familias.value = "";
      codIva.value = "";
      qtdFacturada.value = 0;
      precoUnit.value = "0";
      dataInicio.value = "";
      total.value = 0;
      dataFim.value = "";
      desconto.value = 0;
      artigo.value = "";
      dialog.value = false;

      return;
    }
  } else {
    toast.warning("Data Fim tem de ser superior à Data Inicio", {
      timeout: 3000,
    });
  }
};

// Eliminar
const eliminarLinhaArtigo = (item: any) => {
  const { id } = item.raw;

  const indexToDelete = linhaTabela.value.findIndex(
    (item: any) => item.id === id
  );

  if (indexToDelete >= 0) {
    linhaTabela.value.splice(indexToDelete, 1);
    linhas.value.splice(indexToDelete, 1);
  }
};

const onIntersectArtigos = () => {
  page.value += 1;
};

const buscarArtigos = async () => {};

const buscarCentroCusto = async () => {
  var response: any[] = await UsecaseBuscarCentroCusto.handler(
    `?page=${page.value}&take=${take.value}`
  );
  listaCentroCusto.value = [...listaCentroCusto.value, ...response];
};

const obterListaFamilia = async () => {
  const response = await UsecaseObterFamilias.handler();
  listaFamilias.value = response;
};

const obterListaArtigoFamilia = async (id: string) => {
  const response = await UsecaseObterArtigosFamilias.handler(id);
  listaFamiliaArtigos.value = response.object;

  listaFamiliaArtigos.value.forEach((el: any) => {
    el["full_codigo"] = el["codArt"] + " - " + el["descricao"];
  });
};

const obterDescricaoArtigo = (artigo: any) => {
  listaFamiliaArtigos.value.forEach((item: any) => {
    if (item.codArt === artigo) {
      descricao.value = item.descricao;
      codIva.value = item.iva;
      return;
    }
  });
};

async function handle(index: any) {
  dialogInformacaoArtigo.value = true;
  LinhasArtigo.value = [];
  titleOcupacao.value = `Ocupação - ${index.raw.codArt}`;

  const response = await UsecaseObterListaArtigosEmUrso.handler(
    `?codigoArtigo=${index.raw.codArt}`
  );
  LinhasArtigo.value.push(response.object);
  console.log("LinhasArtigo", LinhasArtigo);
}

function compararDatas() {
  const dataAtual = FormatarData(`${new Date()}`);
  const dataInicioNova = FormatarData(`${new Date(dataInicio.value)}`);

  if (dataInicioNova >= dataAtual) {
    isVerificaDataInicio.value = false;
  } else {
    isVerificaDataInicio.value = true;
    toast.warning("A data de Inicio não pode ser inferior à data de hoje", {
      timeout: 3000,
    });
  }
}

function formatarPrecoUnitario() {
  // Obtém o valor atual do preço unitário
  let valor = `${precoUnit.value}`;

  // Remove qualquer formatação existente (por exemplo, pontos de milhares) e espaços em branco
  valor = valor.replace(/\s+/g, "").replace(/\./g, "");

  // Adiciona pontos de milhares e "kz"
  valor = valor.replace(/\B(?=(\d{3})+(?!\d))/g, ".") + "kz";

  // Define o valor formatado de volta no campo
  precoUnit.value = valor;
}

// Calcular Total
watch([dataFim, dataInicio, sabados.value, domingos.value], () => {});

// watch([precoUnit, desconto, codIva], () => {});

const calcularValorTotal = () => {
  qtdFacturada.value = contarDiasFacturadosEntreDatas(
    new Date(dataInicio.value),
    new Date(dataFim.value),
    sabados.value.value,
    domingos.value.value
  );

  qtdAlg.value = DiferencaData(dataInicio.value, dataFim.value);

  //  Pegar as Taxas
  var taxaIVA = Number(codIva.value);
  var taxaDesconto = Number(desconto.value);

  var precoFormatadoSemVirgula = FormatarPrecoSemPontoVirgula(
    `${precoUnit.value}`
  );

  dataFim.value === "" && dataInicio.value === ""
    ? (qtdFacturada.value = qtdMesesAluguer.value)
    : qtdFacturada.value;

  var valorTotalSemIVA = Number(precoFormatadoSemVirgula) * qtdFacturada.value;
  var valorIVA = valorTotalSemIVA * (taxaIVA / 100);
  var valorTotalComIVA = valorTotalSemIVA + valorIVA;
  var valorDesconto = valorTotalComIVA * (taxaDesconto / 100);

  total.value = FormatarPreco(valorTotalComIVA - valorDesconto);

  return total.value;
};

watchEffect(() => {
  calcularValorTotal();
});

watch([sabados.value, domingos.value], () => {
  linhaTabela.value.forEach((element: any) => {
    let dataInicial = FormatarDataEditavel(element["dataInicio"]);
    let dataFinal = FormatarDataEditavel(element["dataFim"]);

    element["qtdFacturada"] = contarDiasFacturadosEntreDatas(
      new Date(dataInicial),
      new Date(dataFinal),
      sabados.value.value,
      domingos.value.value
    );

    element["quantidade"] = DiferencaData(dataInicial, dataFinal);

    var valorTotalSemIVA =
      Number(FormatarPrecoSemPontoVirgula(element["precoUnit"])) *
      Number(element["qtdFacturada"]);

    var valorIVA = valorTotalSemIVA * (Number(element["codIva"]) / 100);
    var valorTotalComIVA = valorTotalSemIVA + valorIVA;
    var valorDesconto = valorTotalComIVA * (element["desconto"] / 100);

    element["total"] = FormatarPreco(valorTotalComIVA - valorDesconto);
  });
});

watch(familias, () => {
  obterListaArtigoFamilia(familias.value);
});

watch(artigo, () => {
  obterDescricaoArtigo(artigo.value);
});

// Calcular os Meses
watch([mes.value, longa.value, qtdMeses], () => {
  qtdFacturada.value = 31 * Number(qtdMeses.value);
  qtdMesesAluguer.value = qtdFacturada.value;
});

function recalcularTotal(item: any) {
  var valorTotalSemIVA = Number(item.raw.precoUnit) * item.raw.qtdFacturada;
  var valorIVA = valorTotalSemIVA * (item.raw.codIva / 100);
  var valorTotalComIVA = valorTotalSemIVA + valorIVA;
  var valorDesconto = valorTotalComIVA * (item.raw.desconto / 100);
  item.raw.total = valorTotalComIVA - valorDesconto;
}

function changeDatas(item: any) {
  item.raw.qtdFacturada = contarDiasFacturadosEntreDatas(
    new Date(item.raw.dataInicio),
    new Date(item.raw.dataFim),
    sabados.value.value,
    domingos.value.value
  );
  item.raw.qtdAlg = DiferencaData(item.raw.dataInicio, item.raw.dataFim);
  recalcularTotal(item);
}

onMounted(() => {
  obterClientes();
  obterTipoPagamentos();
  obterVendedores();
  obterTipoDocumentos();
  obterNomeDoUtilizador();
  obterListaFamilia();
  buscarCentroCusto();

  sabados.value.value = true;
  domingos.value.value = false;
  feriados.value.value = false;
  norm.value.value = true;
  mes.value.value = false;
  longa.value.value = false;

  conversaoProforma.value = false;
});

watch(client.value, () => {
  clients.value.forEach((item: any) => {
    if (item.entidade === client.value.value) {
      tipoCliente.value = item.tipoEntidade;
    }
  });
});

watch(
  () => storeProforma.dataProforma,
  (newValue, oldValue) => {
    if (newValue !== oldValue) {
      conversaoProforma.value = true;

      vendedor.value.value = storeProforma.dataProforma.raw.vendedorCod;
      aoCuidado.value.value = storeProforma.dataProforma.raw.aoCuidado;
      email.value.value = storeProforma.dataProforma.raw.email;
      tipoCliente.value = storeProforma.dataProforma.raw.tipoEnt;
      client.value.value = storeProforma.dataProforma.raw.entidadeCodigo;
      tipoDoc.value.value = "P";
      telefone.value.value = storeProforma.dataProforma.raw.telefone;
      sabados.value.value = storeProforma.dataProforma.raw.sab;

      linhas.value = [...storeProforma.listaArtigos];
      linhas.value.forEach((item: any) => {
        item["precoUnit"] = FormatarPrecoSemPontoVirgula(item["precoUnit"]);
        item["total"] = FormatarPrecoSemPontoVirgula(item["total"]);
      });

      local.value.value = storeProforma.dataProforma.raw.localTrab;
      condPagamento.value.value = storeProforma.dataProforma.raw.condPagCodigo;
      obs.value = storeProforma.dataProforma.raw.obs;

      linhaTabela.value = [...storeProforma.listaArtigos];
    } else {
      conversaoProforma.value = false;
    }
  }
);

function limparCache() {
  conversaoProforma.value = false;
}
</script>

<template>
  <div class="container mt-4 spacing">
    <form @submit.prevent="addFormPedidoProposta">
      <v-row class="mb-2">
        <v-col align="right">
          <v-btn color="red" class="mr-2" @click="limparFormulario"
            >Limpar Formulário</v-btn
          >
          <v-btn type="submit" class="btnSubmit">Salvar</v-btn>
        </v-col>
      </v-row>
      <v-row>
        <v-col cols="12" md="4">
          <v-autocomplete
            clearable
            label="Documento *"
            v-model="tipoDoc.value.value"
            :error-messages="tipoDoc.errorMessage.value"
            :items="tipoDocumento"
            item-title="descTipo"
            item-value="tipo"
          >
          </v-autocomplete>
        </v-col>

        <v-col cols="12" md="4">
          <v-autocomplete
            clearable
            label="Cliente *"
            v-model="client.value.value"
            :items="clients"
            :error-messages="client.errorMessage.value"
            item-title="nome_codigo"
            item-value="entidade"
            @input="obterClientes"
          >
            <template v-slot:append-item>
              <div v-intersect="onIntersect" class="pa-4 teal--text">
                Buscando mais clientes ...
              </div>
            </template>
          </v-autocomplete>
        </v-col>

        <v-col cols="12" md="2">
          <v-text-field
            v-model="tipoCliente"
            label="Tipo Cliente *"
            readonly
          ></v-text-field>
        </v-col>

        <v-col cols="12" md="2">
          <v-text-field
            v-model="dataDocumento"
            label="Data Doc"
            readonly
          ></v-text-field>
        </v-col>

        <v-col cols="12" md="4">
          <v-autocomplete
            clearable
            label="Cond. Pagamento *"
            item-title="descricao"
            item-value="codigo"
            v-model="condPagamento.value.value"
            :error-messages="condPagamento.errorMessage.value"
            :items="tiposPagamentos"
          >
          </v-autocomplete>
        </v-col>

        <v-col cols="12" md="4">
          <v-autocomplete
            clearable
            v-model="vendedor.value.value"
            :error-messages="vendedor.errorMessage.value"
            :items="vendedores"
            item-title="nome"
            item-value="codigo"
            label="Vendedor *"
          >
          </v-autocomplete>
        </v-col>

        <v-col cols="12" md="4">
          <v-text-field
            v-model="aoCuidado.value.value"
            label="Ao cuidado *"
            :error-messages="aoCuidado.errorMessage.value"
          ></v-text-field>
        </v-col>

        <v-col cols="12" md="4">
          <v-text-field
            v-model="email.value.value"
            label="E-mail *"
            variant="outlined"
            :error-messages="email.errorMessage.value"
          ></v-text-field>
        </v-col>

        <v-col cols="12" md="4">
          <v-text-field
            v-model="telefone.value.value"
            label="Telefone *"
            :maxlength="9"
            :error-messages="telefone.errorMessage.value"
          ></v-text-field>
        </v-col>

        <v-col cols="12" md="4">
          <v-text-field
            v-model="local.value.value"
            label="Local de aluguer *"
            :error-messages="local.errorMessage.value"
          ></v-text-field>
        </v-col>
      </v-row>

      <v-divider class="mb-6 mt-3"></v-divider>

      <v-row
        calign-content="center"
        align="center"
        justify="start"
        justify-content="start"
      >
        <v-col cols="6">
          Tipo de Aluguer:
          <div class="checkboox">
            <div class="aluguerLinha">
              <v-radio-group v-model="controlarAluguer" inline>
                <v-radio
                  v-for="(item, index) in aluguerItem"
                  :key="index"
                  :label="item"
                  :value="item"
                  @change="handleChange(item)"
                ></v-radio>
              </v-radio-group>

              <v-expand-transition>
                <div key="aluguer" v-if="aluguer" class="form-control">
                  <label for="aluguer">Meses <sup>*</sup></label>

                  <input type="number" v-model="qtdMeses" class="inputMes" />
                </div>
              </v-expand-transition>
            </div>
          </div>
        </v-col>
        <v-col cols="5">
          Faturar:
          <div class="checkboox">
            <v-checkbox
              type="checkbox"
              v-model="sabados.value.value"
              :value="true"
              label="Sábados"
            >
            </v-checkbox>
            <v-checkbox
              type="checkbox"
              v-model="domingos.value.value"
              :value="true"
              label="Domingos"
            >
            </v-checkbox>
            <v-checkbox
              type="checkbox"
              v-model="feriados.value.value"
              :value="true"
              label="Feriados"
            >
            </v-checkbox>
          </div>
        </v-col>
      </v-row>

      <v-divider class="mb-6 mt-3"></v-divider>

      <v-row>
        <v-col cols="12">
          <v-textarea
            variant="outlined"
            v-model="obs"
            label="Observações"
          ></v-textarea>
        </v-col>
      </v-row>
    </form>

    <v-row>
      <v-col cols="12" md="12">
        <v-divider class="mb-2 mt-1"></v-divider>
        <h4>Linha Artigo</h4>
        <v-divider class="mb-2 mt-2"></v-divider>
      </v-col>

      <v-col cols="12" md="12">
        <v-text-field
          v-model="search"
          append-inner-icon="mdi-magnify"
          label="Search"
          single-line
          hide-details
        ></v-text-field>

        <v-data-table
          :headers="headers"
          :items="linhaTabela"
          class="overflow-x-auto"
          :search="search"
        >
          <template v-slot:[`item.accoes`]="{ item }">
            <v-menu transition="slide-x-transition">
              <template v-slot:activator="{ props }">
                <v-btn
                  icon="mdi-dots-vertical"
                  size="small"
                  v-bind="props"
                  class="color-actions"
                >
                </v-btn>
              </template>
              <v-list>
                <v-list-item :value="0">
                  <v-list-item-title @click="eliminarLinhaArtigo(item)"
                    >Eliminar Linha</v-list-item-title
                  >
                </v-list-item>
              </v-list>
            </v-menu>
          </template>

          <template
            v-slot:[`item.dataInicio`]="{ item }"
            v-if="conversaoProforma"
          >
            <v-col class="larguraInputData">
              <v-text-field
                class="mt-2"
                type="date"
                v-model="item.raw.dataInicio"
                @input="changeDatas(item)"
              ></v-text-field>
            </v-col>
          </template>

          <template v-slot:[`item.dataFim`]="{ item }" v-if="conversaoProforma">
            <v-col class="larguraInputData">
              <v-text-field
                class="mt-2"
                type="date"
                v-model="item.raw.dataFim"
                @input="changeDatas(item)"
              ></v-text-field>
            </v-col>
          </template>

          <template
            v-slot:[`item.qtdFacturada`]="{ item }"
            v-if="conversaoProforma"
          >
            <td>{{ item.raw.qtdFacturada }}</td>
          </template>
          <template
            v-slot:[`item.quantidade`]="{ item }"
            v-if="conversaoProforma"
          >
            <td>{{ item.raw.quantidade }}</td>
          </template>
          <template
            v-slot:[`item.precoUnit`]="{ item }"
            v-if="conversaoProforma"
          >
            <v-col class="larguraInputData">
              <v-text-field
                v-model="item.raw.precoUnit"
                variant="outlined"
                :value="item.raw.precoUnit"
                @input="recalcularTotal(item)"
                class="preco"
              ></v-text-field>
            </v-col>
          </template>
          <template
            v-slot:[`item.desconto`]="{ item }"
            v-if="conversaoProforma"
          >
            <v-col class="larguraInputDesconto">
              <v-text-field
                type="number"
                v-model="item.raw.desconto"
                variant="outlined"
                min="0"
                max="100"
                size="3"
                @input="recalcularTotal(item)"
              ></v-text-field>
            </v-col>
          </template>
          <template v-slot:[`item.total`]="{ item }" v-if="conversaoProforma">
            <td class="preco">{{ item.raw.total }}</td>
          </template>
        </v-data-table>

        <!-- Modal -->
        <div class="text-center">
          <v-dialog v-model="dialog" width="90%">
            <template v-slot:activator="{ props }">
              <v-col align="right">
                <v-btn color="primary" @click="limparCache" v-bind="props"
                  >Adicionar</v-btn
                >
              </v-col>
            </template>

            <v-card>
              <TitleModal title="Adicionar Artigos" @fechar="dialog = false" />

              <form @submit.prevent="addLinhaPedidoProposta">
                <v-card-text>
                  <div class="spacing">
                    <v-row>
                      <v-col cols="12" sm="6" md="2">
                        <v-autocomplete
                          label="Famílias *"
                          :items="listaFamilias"
                          v-model="familias"
                          item-title="descricao"
                          item-value="codigo"
                        >
                        </v-autocomplete>
                      </v-col>
                      <v-col cols="12" sm="6" md="4">
                        <v-autocomplete
                          label="Artigo *"
                          :items="listaFamiliaArtigos"
                          v-model="artigo"
                          item-title="full_codigo"
                          item-value="codArt"
                          item-disabled="disabled"
                          @input="buscarArtigos"
                        >
                          <template v-slot:item="{ props, item }">
                            <v-list-item v-bind="props">
                              <template v-slot:append>
                                <v-btn
                                  color="success"
                                  icon="mdi-circle"
                                  variant="text"
                                  v-if="item.raw.status === 'DISP'"
                                ></v-btn>
                                <v-btn
                                  v-else-if="item.raw.status === 'AWAIT'"
                                  color="yellow"
                                  icon="mdi-circle"
                                  variant="text"
                                  @click="handle(item)"
                                ></v-btn>
                                <v-btn
                                  v-else-if="item.raw.status === 'ALUG'"
                                  color="red"
                                  icon="mdi-circle"
                                  variant="text"
                                  @click="handle(item)"
                                ></v-btn>
                                <v-btn
                                  v-else
                                  color="success"
                                  icon="mdi-circle"
                                  variant="text"
                                ></v-btn>
                              </template>
                            </v-list-item>
                          </template>
                          <template v-slot:append-item>
                            <div
                              v-intersect="onIntersectArtigos"
                              class="pa-4 teal--text"
                            >
                              Buscando mais artigos...
                            </div>
                          </template>
                        </v-autocomplete>
                      </v-col>

                      <v-col cols="2">
                        <v-text-field
                          label="Data de Início"
                          type="date"
                          v-model="dataInicio"
                          variant="outlined"
                          @blur="compararDatas"
                          :disabled="
                            mes.value.value === true ||
                            longa.value.value === true
                          "
                        ></v-text-field>
                      </v-col>
                      <v-col cols="2">
                        <v-text-field
                          label="Data de Fim"
                          type="date"
                          v-model="dataFim"
                          variant="outlined"
                          :disabled="
                            mes.value.value === true ||
                            longa.value.value === true
                          "
                        ></v-text-field>
                      </v-col>
                      <v-col cols="2">
                        <v-text-field
                          label="Quantidade Aluguer"
                          type="number"
                          readonly
                          v-model="qtdAlg"
                          variant="outlined"
                        ></v-text-field>
                      </v-col>
                      <v-col cols="2">
                        <v-text-field
                          label="Quantidade Facturada"
                          type="number"
                          readonly
                          v-model="qtdFacturada"
                          variant="outlined"
                        ></v-text-field>
                      </v-col>
                      <v-col cols="2">
                        <v-text-field
                          label="Preço Unitário"
                          variant="outlined"
                          :value="precoUnit"
                          v-model="precoUnit"
                          @blur="formatarPrecoUnitario"
                          class="preco"
                        ></v-text-field>
                      </v-col>

                      <v-col cols="12" sm="6" md="2">
                        <v-text-field
                          label="Desconto"
                          type="number"
                          v-model="desconto"
                          variant="outlined"
                          min="0"
                          max="100"
                          size="3"
                        ></v-text-field>
                      </v-col>
                      <v-col cols="12" sm="6" md="2">
                        <v-text-field
                          label="IVA"
                          type="text"
                          readonly
                          v-model="codIva"
                          variant="outlined"
                        ></v-text-field>
                      </v-col>
                      <v-col cols="12" sm="6" md="2">
                        <v-autocomplete
                          label="Centro de Custo"
                          :items="listaCentroCusto"
                          v-model="centroCusto"
                          item-title="descricao"
                          item-value="centro"
                          @input="buscarCentroCusto"
                        >
                          <template v-slot:append-item>
                            <div
                              v-intersect="onIntersectCentroCusto"
                              class="pa-4 teal--text"
                            >
                              Buscando mais centros...
                            </div>
                          </template>
                        </v-autocomplete>
                      </v-col>
                      <v-col cols="12" sm="6" md="2">
                        <v-text-field
                          label="Total"
                          type="text"
                          readonly
                          v-model="total"
                          variant="outlined"
                          class="preco"
                        ></v-text-field>
                      </v-col>
                    </v-row>
                    <v-row>
                      <v-col align="right">
                        <v-btn
                          type="submit"
                          :disabled="isVerificaDataInicio"
                          :class="`${isVerificaDataInicio ? '' : 'btnSubmit'}`"
                          :color="`${isVerificaDataInicio && 'red'}`"
                          >Adicionar Linha</v-btn
                        >
                      </v-col>
                    </v-row>
                  </div>
                </v-card-text>
              </form>
            </v-card>
          </v-dialog>
        </div>
        <!-- Modal -->

        <!-- Informações do artigo -->
        <v-dialog v-model="dialogInformacaoArtigo" width="35%">
          <v-card>
            <TitleModal
              :title="titleOcupacao"
              @fechar="dialogInformacaoArtigo = false"
            />

            <div class="">
              <v-container>
                <v-table>
                  <thead>
                    <tr>
                      <th class="text-left">Cliente</th>
                      <th class="text-left">Data Entrega</th>
                      <th class="text-left">Data Recolha</th>
                    </tr>
                  </thead>
                  <tbody>
                    <template v-if="LinhasArtigo.length > 0">
                      <tr v-for="(item, index) in LinhasArtigo" :key="index">
                        <td>{{ item.clienteNome }}</td>
                        <td>{{ FormatarData(item.dataInicial) }}</td>
                        <td>{{ FormatarData(item.dataFinal) }}</td>
                      </tr>
                    </template>
                    <template v-else>
                      <tr>
                        <td colspan="4" class="text-center">Podes usar este</td>
                      </tr>
                    </template>
                  </tbody>
                </v-table>
              </v-container>
            </div>
          </v-card>
        </v-dialog>
      </v-col>
    </v-row>
  </div>
</template>

<style lang="scss" scoped></style>
