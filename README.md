# Integração: Wazuh e Clamav (anti-malware for linux-based system)
Neste laboratório, configurei a integração do clamav com o wazuh e ganhei experencia sobre gerenciamento de permissões avançadas no Ubuntu 24.04. Usando imagens demostro a prova dos nove, referente a experiência vivada neste laboratório.


## Hoje vou apresentar-vos uma ferramenta de gereciamento de permissões e acesso que todo administrador de cibersegurança ou estudante de cibersegurança/segurança da informação deve saber.

Ao configurar a integração do clamav (anti-malware for Linux) com o Wazuh Manager me deparei com uma situação que envolve muito sobre gerenciamentos de permissões. O clamav é um suite de anti-malware para sistemas baseado no kernel Linux, essa suite é composta por clamdscan, freshclam e clamscan. O clamdscan requer privilegio execute (x) e write (w) para executar uma solicitação de escanear para o serviço clamd via socket, o database das assinaturas fica na memoria e o clamd realizar o escanear ou varredura para os directory e files que foram solicitados previamente pelo o cliente (clamdscan), pense como; anti-malware já estava ligado/ativo e apenas realizou-se a varredura, melhor do que carregar todo o database de assinaturas para cada scan, isso que o clamscan faz. Se/quando realizar 6 scan usando o clamscan, ele [clamascan] vai carregar todo database nas 6 vezes.

Tentei realizar a varredura na pasta /home/naftali/ as permissões da pastas naftali/ são "drwx--x---" isso quer dizer que nenhum outro grupo ou usuário pode ler e executar arquivos dentro da pasta "naftali/", a primeira coisa que veio na minha cabeça era vou executar um chmod 775 naftali e continuar o meu laboratório virtual, mas depois me perguntei e se fosse no servidor do banco? da empresa? o que eu faria? Surgiu a curiosidade de buscar e implantar uma solução mais madura (privilégio mínimo) - lembrei das perguntas do Mr. Dony Bellino.

Depois de 5 minutos de pesquisa encontrei um software no Ubuntu 24, a solução é "acl" ou em português lista de controle de acesso. Bora instalar? "apt install acl -y". Para definir a "ACE" ou "entradas de lista de acesso" deve-se usar o  comando "setfacl" e para visualizar as ACE de uma pasta ou arquivo deve-se usar o comando getfacl.

Manual em en-us, o inglês maximizou o entendimento e economizou segundos de tradução. O parâmetro "-R" é para recursividade, o "-d" aplica as permissões em arquivos ou pasta criados futuramente. Sem o parâmetro "-d" a regra comporta-se de forma efetiva ou seja aplica as permissões para os arquivos ou pastas existentes dentro da pasta "naftali/" e for fim o "-m" de modificar as permissões da pasta "naftali/".

O objetivo é permitir com que a ferramenta clamdscan leia e execute as pastas e arquivos dentro da pasta para poder completar a varredura. E se um arquivo/pasta não tem a permissão x, como a pasta **home/naftali/.ssh/** ou o arquivo **home/naftali/.bash_history**, a ferramenta clamd não deve executar. 


## Verificação de permissões e erro.

- A imagem abaixo mostra o momento em que eu listei as pastas/arquivos na raiz do sistema de arquivo usando comando **ls -l**
- As permissões da pasta "home" continua as mesmas.
- Quando tentei executar a ferramenta clamdscan, ocorreu uma falha na verificação do caminho até a pasta destino, especificamente permissão negada.
  
<img width="1072" height="725" alt="1" src="https://github.com/user-attachments/assets/6fab50ee-e48a-489c-a590-026ba2100c92" />


## Instalação da ferramenta acl via apt.

- Para instalar a ferramenta use  o comanod "sudo apt install acl -y"
- A razão do erro, foi porque eu executei a ferramenta apt com baixo privilégio, ou seja, sem o sudo.

<img width="1286" height="395" alt="2" src="https://github.com/user-attachments/assets/321985c8-3619-423c-8d34-3e6883cf6aa3" />

## Definindo ACE - Access Control Entry.

```
1 -  naftali@ubuntu:/home setfacl -R -d -m u:clamav:r-X naftali/ - comportamento padrão ou seja a regra será aplicada apenas aos novos arquivos e pastas.
Por meio do  parâmetro X (maiusculo) a ferramenta não executar arquivos/pastas que tenham permissões iguais a pasta "/home/naftali/.ssh/".
```
```
2 -  naftali@ubuntu:/home setfacl -R -m u:clamav:r-X naftali/ - Para já, aplica a regra em arquivos/pastas já existentes. 
3 -  naftali@ubuntu:/home setfacl -R -m m:r-X naftali/ ->> Defini a mask para impedir atribuição da permisão escrita (w) via setfacl -R -m u:clamav:rwX naftali/ 
```

- Depois de difinir as regras, usei o comanodo getfacl para ver o status das permissões na pasta naftali/.
- OBS: Dentro da pasta naftali tem dois arquivos do https://www[.]eicar[.]org
- Em seguida, tentei novamente executar o comando **clamdscan naftali/** e funcionou.
- Dois arquivos com a assinatura do eicar foram encontrados na pasta **naftali**.
- Como a intergrão com o clamav já foi feita e o agente wazuh já enstá instalado e ativado. O wazuh vai mostra uma alerta nos eventos de segurança.  
>
<img width="599" height="598" alt="3" src="https://github.com/user-attachments/assets/3182f77c-86d8-48b6-9939-6c62ef86a6d1" />

## Wazuh - Security events
- A imagem abaixo mostra 3 eventos de segurança, um evento 1 com nivél de regra 3 e dois eventos com o nivel de regra 8.
- Os eventos de segurança com o id de regra *52502**, tem uma descrição referente a detecção de vírus por meio do suite clamav.
- O evento de segurança com o nivél de regra 3 está relacionado ao modulo de autenticação do sistema Ubuntu, provavelmente uma sessão root foi fechada.
>
<img width="1365" height="637" alt="4" src="https://github.com/user-attachments/assets/9d40d5a5-34ed-46de-8219-4fc270574d06" />

## Detalhes - Clamav: Virus detected

- As imagens abaixo fornem mais informações detalhadas como id do agente, nome do agente, caminho completo arquivo detectado como vírus, descrição da regra e horario da detecção.

>
<img width="1364" height="642" alt="5" src="https://github.com/user-attachments/assets/a2ea7b51-faa8-40e3-aa5f-dfa93b2d1c15" />
>
<img width="1362" height="593" alt="6" src="https://github.com/user-attachments/assets/a6a979a3-e08a-4ec5-915f-f278cc9b2446" />

Resources:
Comunidade Ubuntu: https://help.ubuntu.com/community/FilePermissionsACLs
Integração wazuh com o clamav: https://documentation.wazuh.com/current/user-manual/capabilities/malware-detection/clam-av-logs-collection.html
