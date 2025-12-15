# src/dominio/erros.py

class ErroDeDominio(Exception):
    """Exceção base para erros de regras de negócio."""
    pass

class ConflitoDeHorario(ErroDeDominio):
    """Erro levantado quando um agendamento conflita com outro existente."""
    pass

# src/dominio/modelos.py
from dataclasses import dataclass, field
from datetime import datetime, timedelta

# IDs simples para o exemplo
proximo_id_consulta = 1 

def gerar_id_consulta():
    global proximo_id_consulta
    _id = proximo_id_consulta
    proximo_id_consulta += 1
    return _id

@dataclass(frozen=True) # Entidades do domínio
class Medico:
    id: int
    nome: str
    especialidade: str

@dataclass(frozen=True)
class Paciente:
    id: int
    nome: str

@dataclass
class Consulta:
    medico: Medico
    paciente: Paciente
    data_hora: datetime
    duracao_minutos: int = 30
    id: int = field(default_factory=gerar_id_consulta, init=False)

    @property
    def fim(self) -> datetime:
        """Calcula o horário de término da consulta."""
        return self.data_hora + timedelta(minutes=self.duracao_minutos)

    def conflita_com(self, outra_consulta: 'Consulta') -> bool:
        """Regra de Negócio: Verifica se há conflito de horário para o mesmo médico."""
        
        # 1. Deve ser o mesmo médico para haver conflito de agenda
        if self.medico.id != outra_consulta.medico.id:
            return False

        # 2. Verifica se os períodos se sobrepõem
        inicio1, fim1 = self.data_hora, self.fim
        inicio2, fim2 = outra_consulta.data_hora, outra_consulta.fim

        # Conflito se o início de um está dentro do período do outro, ou vice-versa
        sobreposicao = (inicio1 < fim2) and (inicio2 < fim1)
        
        return sobreposicao
    
    
# src/dominio/fabricas.py

# src/dominio/repositorios.py
from abc import ABC, abstractmethod
from typing import List, Optional
from .modelos import Consulta

class RepositorioDeConsultas(ABC):
    """Define a interface (o 'contrato') para a persistência de Consultas."""
    
    @abstractmethod
    def salvar(self, consulta: Consulta) -> None:
        """Salva ou atualiza uma consulta."""
        pass
        
    @abstractmethod
    def buscar_por_medico(self, medico_id: int) -> List[Consulta]:
        """Busca todas as consultas agendadas para um médico."""
        pass

    @abstractmethod
    def buscar_por_id(self, consulta_id: int) -> Optional[Consulta]:
        """Busca uma consulta pelo seu ID."""
        pass
    
    # src/infra/repositorios.py
from typing import List, Dict, Optional
from src.dominio.repositorios import RepositorioDeConsultas
from src.dominio.modelos import Consulta

class RepositorioDeConsultasEmMemoria(RepositorioDeConsultas):
    """Implementação concreta (em memória) do repositório de consultas."""

    def __init__(self, consultas_iniciais: Optional[List[Consulta]] = None):
        # Armazena consultas em um dicionário onde a chave é o ID da consulta
        self.consultas: Dict[int, Consulta] = {}
        if consultas_iniciais:
            for c in consultas_iniciais:
                self.consultas[c.id] = c
                
    def salvar(self, consulta: Consulta) -> None:
        """Salva a consulta. Se o ID já existe, atualiza."""
        self.consultas[consulta.id] = consulta
        
    def buscar_por_medico(self, medico_id: int) -> List[Consulta]:
        """Busca consultas de um médico específico."""
        return [
            consulta for consulta in self.consultas.values() 
            if consulta.medico.id == medico_id
        ]

    def buscar_por_id(self, consulta_id: int) -> Optional[Consulta]:
        """Busca uma consulta pelo seu ID."""
        return self.consultas.get(consulta_id)
        
    def buscar_todos(self) -> List[Consulta]:
        """Método auxiliar para testes e UI."""
        return list(self.consultas.values())
    

    
  
  # src/app/fachada.py
from src.dominio.repositorios import RepositorioDeConsultas
from src.dominio.modelos import Consulta, Medico, Paciente
from src.dominio.erros import ConflitoDeHorario, ErroDeDominio # Importe ErroDeDominio genérico
from datetime import datetime
from typing import List

class FachadaDeAgendamento:
    def __init__(self, repositorio_consultas: RepositorioDeConsultas):
        self._repo = repositorio_consultas

    # ... (método agendar_consulta e listar_consultas_medico mantidos iguais) ...

    def cancelar_consulta(self, consulta_id: int) -> None:
        """
        Cancela uma consulta existente.
        """
        consulta = self._repo.buscar_por_id(consulta_id)
        if not consulta:
            raise ErroDeDominio("Consulta não encontrada para cancelamento.")
        
        # Aqui poderíamos ter regras de negócio (ex: não cancelar se for no passado)
        # Mas para o trabalho, vamos direto à remoção:
        self._repo.remover(consulta_id)

    def reagendar_consulta(self, consulta_id: int, nova_data_str: str) -> Consulta:
        """
        Altera a data de uma consulta existente, verificando novos conflitos.
        """
        # 1. Verifica se a consulta existe
        consulta_atual = self._repo.buscar_por_id(consulta_id)
        if not consulta_atual:
            raise ErroDeDominio("Consulta não encontrada para reagendamento.")

        # 2. Parse da nova data
        try:
            nova_data_hora = datetime.strptime(nova_data_str, "%Y-%m-%d %H:%M")
        except ValueError:
            raise ValueError("Formato de data inválido. Use YYYY-MM-DD HH:MM.")

        # 3. Cria uma "cópia temporária" da consulta com a nova data para testar conflito
        # Importante: Criamos um objeto novo apenas para testar a regra de conflito
        consulta_teste = Consulta(
            medico=consulta_atual.medico,
            paciente=consulta_atual.paciente,
            data_hora=nova_data_hora,
            duracao_minutos=consulta_atual.duracao_minutos
        )

        # 4. Verifica conflitos com OUTRAS consultas do mesmo médico
        consultas_do_medico = self._repo.buscar_por_medico(consulta_atual.medico.id)
        
        for outra in consultas_do_medico:
            # PULO DO GATO: Não podemos verificar conflito com ela mesma (o registro antigo)
            if outra.id == consulta_atual.id:
                continue
                
            if consulta_teste.conflita_com(outra):
                raise ConflitoDeHorario(
                    f"Conflito no reagendamento! O médico já tem agenda ocupada neste horário."
                )

        # 5. Se passou pelos testes, atualiza o objeto original e salva
        consulta_atual.data_hora = nova_data_hora
        self._repo.salvar(consulta_atual)
        
        return consulta_atual
    
    # main.py
import sys
from datetime import datetime

# Ajusta o caminho para que o Python encontre os módulos em 'src'
sys.path.append('src')

from src.dominio.modelos import Medico, Paciente
from src.infra.repositorios import RepositorioDeConsultasEmMemoria
from src.app.fachada import FachadaDeAgendamento
from src.dominio.erros import ErroDeDominio

# --- Dados Iniciais (Simulação de um Cadastro Pré-existente) ---
medicos = [
    Medico(id=1, nome="Dra. Ana Costa", especialidade="Cardiologia"),
    Medico(id=2, nome="Dr. Pedro Silva", especialidade="Pediatria"),
]

pacientes = [
    Paciente(id=101, nome="João Santos"),
    Paciente(id=102, nome="Maria Oliveira"),
]

# --- Injeção de Dependência (Montagem do Sistema) ---
# 1. Cria a implementação do Repositório (Infra)
repo_consultas = RepositorioDeConsultasEmMemoria()

# 2. Cria a Fachada (Aplicação), injetando o Repositório
fachada = FachadaDeAgendamento(repo_consultas)

# --- Lógica de UI (Linha de Comando) ---

def exibir_medicos():
    print("\n--- Médicos Disponíveis ---")
    for m in medicos:
        print(f"ID: {m.id} | Nome: {m.nome} | Especialidade: {m.especialidade}")

def exibir_pacientes():
    print("\n--- Pacientes Cadastrados ---")
    for p in pacientes:
        print(f"ID: {p.id} | Nome: {p.nome}")

def menu_agendar():
    exibir_medicos()
    try:
        medico_id = int(input("Informe o ID do Médico: "))
        medico = next((m for m in medicos if m.id == medico_id), None)
        if not medico:
            print("Médico não encontrado.")
            return

        exibir_pacientes()
        paciente_id = int(input("Informe o ID do Paciente: "))
        paciente = next((p for p in pacientes if p.id == paciente_id), None)
        if not paciente:
            print("Paciente não encontrado.")
            return

        data_hora_str = input("Data e Hora (YYYY-MM-DD HH:MM): ")
        
        # A UI chama a Fachada, delegando toda a lógica
        nova_consulta = fachada.agendar_consulta(medico, paciente, data_hora_str)
        
        print("\n✅ Consulta agendada com sucesso!")
        print(f"ID: {nova_consulta.id} | Médico: {nova_consulta.medico.nome} | Hora: {nova_consulta.data_hora.strftime('%d/%m %H:%M')}")

    except ErroDeDominio as e:
        print(f"\n❌ ERRO DE AGENDAMENTO: {e}")
    except ValueError as e:
        print(f"\n❌ ERRO DE ENTRADA: {e}")
    except Exception as e:
        print(f"\n❌ Erro inesperado: {e}")

def menu_listar_consultas():
    exibir_medicos()
    try:
        medico_id = int(input("Informe o ID do Médico para listar a agenda: "))
        medico = next((m for m in medicos if m.id == medico_id), None)
        if not medico:
            print("Médico não encontrado.")
            return

        consultas = fachada.listar_consultas_medico(medico_id)
        
        print(f"\n--- Agenda do Dr(a). {medico.nome} ---")
        if not consultas:
            print("Nenhuma consulta agendada.")
            return
            
        for c in sorted(consultas, key=lambda x: x.data_hora):
            hora_inicio = c.data_hora.strftime('%H:%M')
            hora_fim = c.fim.strftime('%H:%M')
            print(f"ID: {c.id} | {c.data_hora.strftime('%d/%m/%Y')} | {hora_inicio}-{hora_fim} | Paciente: {c.paciente.nome}")

    except ValueError:
        print("\n❌ ID inválido.")

def main_menu():
    while True:
        print("\n" + "="*30)
        print("SISTEMA DE AGENDAMENTO MÉDICO")
        print("="*30)
        print("1. Agendar Nova Consulta")
        print("2. Listar Consultas por Médico")
        print("3. Sair")
        
        escolha = input("Escolha uma opção: ")

        if escolha == '1':
            menu_agendar()
        elif escolha == '2':
            menu_listar_consultas()
        elif escolha == '3':
            print("Saindo do sistema. Adeus!")
            break
        else:
            print("Opção inválida.")

if __name__ == '__main__':
    main_menu()
    
    
# tests/test_infra.py
import pytest
from datetime import datetime
from src.dominio.modelos import Consulta, Medico, Paciente
from src.infra.repositorios import RepositorioDeConsultasEmMemoria

# Dados de teste
medico_teste = Medico(id=1, nome="Dr. Repo", especialidade="Teste")
paciente_teste = Paciente(id=1, nome="P. Repo")
data_teste = datetime(2026, 2, 1, 9, 0)

def test_salvar_e_buscar_por_id():
    """Deve salvar uma consulta e recuperá-la pelo ID."""
    repo = RepositorioDeConsultasEmMemoria()
    consulta_original = Consulta(medico_teste, paciente_teste, data_teste)
    
    repo.salvar(consulta_original)
    
    consulta_recuperada = repo.buscar_por_id(consulta_original.id)
    
    assert consulta_recuperada is not None
    assert consulta_recuperada.medico.nome == "Dr. Repo"
    assert consulta_recuperada.paciente.nome == "P. Repo"
    
def test_buscar_por_medico_filtra_corretamente():
    """Deve retornar apenas as consultas do médico solicitado."""
    repo = RepositorioDeConsultasEmMemoria()
    medico2 = Medico(id=2, nome="Dr. Dois", especialidade="Teste")
    
    consulta1 = Consulta(medico_teste, paciente_teste, data_teste)
    consulta2 = Consulta(medico2, paciente_teste, data_teste)
    consulta3 = Consulta(medico_teste, paciente_teste, data_teste.replace(hour=10))
    
    repo.salvar(consulta1)
    repo.salvar(consulta2)
    repo.salvar(consulta3)
    
    consultas_medico1 = repo.buscar_por_medico(medico_teste.id)
    
    assert len(consultas_medico1) == 2
    assert all(c.medico.id == medico_teste.id for c in consultas_medico1)
