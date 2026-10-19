# hardening-linux

![Status](https://img.shields.io/badge/status-em%20desenvolvimento-C9A84C)
![Licença MIT](https://img.shields.io/badge/licen%C3%A7a-MIT-green)

**[English below](#english)**

Role **Ansible** para **endurecer servidores Ubuntu** de forma repetível — e que explica, no próprio README, o que cada controle faz e por que ele existe.

## Por que este projeto

A maioria dos servidores expostos na internet recebe milhares de tentativas de login por dia. Endurecer à mão, servidor por servidor, é lento e esquece passos. Aqui tudo é declarado, testado em contêiner a cada mudança e pode rodar primeiro em **modo auditoria**, que só relata o que está fora do padrão sem alterar nada.

## O que está a caminho

O projeto está sendo construído em entregas pequenas, cada uma com testes e CI. Acompanhe pelos commits e pull requests.

- [ ] SSH endurecido, fail2ban e atualizações automáticas de segurança
- [ ] Firewall (ufw) com perfis prontos
- [ ] Parâmetros de kernel e rede, e auditd
- [ ] Modo auditoria: relatório de conformidade sem alterar o servidor

## Tecnologias

Ansible · Molecule + Docker · ansible-lint · GitHub Actions

## Autor

**Euller Pinheiro**, Diretor Técnico da [ESX Tecnologia](https://esxtecnologia.com) — desenvolvimento de sistemas, infraestrutura e TI gerenciada em São Paulo.

---

<a name="english"></a>
## English

Ansible role to harden Ubuntu servers (SSH, fail2ban, firewall, kernel, auditd), tested with Molecule, with an audit-only mode that reports compliance without changing anything. Work in progress — built in small, tested increments. Licensed under [MIT](LICENSE).
