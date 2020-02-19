
# Continuando API do GoBarber

## Aula 18 - Configurando Multer

### Upload de imagem

Usuário seleciona a imagem, o upload já é feito e o servidor retorna o ID da imagem.
E no json no cadastro de usuário por exemplo, envia o ID da imagem.

Utilizando o [multer](https://github.com/expressjs/multer) para upload de arquivos.

Quando precisa enviar imagem para o servidor, tem que ser como `Multpart-data` (Multpart Form) e não `json`.

Instalando o `multer`: 

`yarn add multer`

Criar uma pasta fora do `src`, para armazenar as imagens: `tmp/uploads`, dentro da pasta `tmp` criar outra pasta `uploads`, onde vai ficar os arquivos físicos de uploads de arquivos.

Criar um arquivo de configuração `multer.js` de dentro da `config`.

```
import multer from 'multer';
import crypto from 'crypto';
import { extname, resolve } from 'path';

export default {
  storage: multer.diskStorage({
	// Local onde o arquivo será salvo na máquina do servidor
    destination: resolve(__dirname, '..', '..', 'tmp', 'uploads'),
    // Gerando o nome da imagem como um hash usando a lib nativa do node: crypto
    filename: (req, file, cb) => {
      crypto.randomBytes(16, (err, res) => {
        if (err) return cb(err);
        return cb(null, res.toString('hex') + extname(file.originalname));
      });
    },
  }),
};
```

Depois criar um rota:

```
import multer from  'multer';
const upload =  multer(multerConfig);

const upload =  multer(multerConfig);

routes.post('/files', upload.single('file'), (req, res) => {
	return res.json({ ok:  true });
});
```

A rota tem que usar o método post, e o corpo da requisição tem que ser um `multpart-form` em vez de `json`.

Depois adicionar um atributo `file` e adicionar o arquivo nesse atributo.

`upload.single('file')` significa que vou enviar apenas um arquivo dentro da propriedade `file`. 

Essa lib multer permite envio de multiplos arquivos.

## Aula 19 - Avatar do Usuário

### Salvando informações do arquivo na base de dados

O atríbuto `req`tem disponível a variável `file` do upload de arquivos que armazena algumas informações sobre o arquivo de upload:

```
{
  "fieldname": "file",
  "originalname": "code-hoc.png",
  "encoding": "7bit",
  "mimetype": "image/png",
  "destination": "/Users/xxx/Developer/bootcamp_rocketseat_studies/gobarber/tmp/uploads",
  "filename": "1d05508938b533ef539026149c597ed5.png",
  "path": "/Users/xxx/Developer/bootcamp_rocketseat_studies/gobarber/tmp/uploads/1d05508938b533ef539026149c597ed5.png",
  "size": 115050
}
```

originalname: é o nome original do arquivo que estava na máquina cliente, que fez o upload.
filename: é o novo nome da imagem que vai ficar salva no servidor.

Para lidar melhor com o upload de arquivo, vou criar o `FileController.js` que conterá a lógica para salvar no banco de dados as referências dos arquivos de upload.

Para salvar os dados do arquivo, vamos criar a tabela files no banco de dados, criando o arquivo de migration.

```
yarn sequelize migration:create --name=create-files
```

E terminar de configurar:

```
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.createTable('files', {
      id: {
        type: Sequelize.INTEGER,
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
      },
      name: {
        type: Sequelize.STRING,
        allowNull: false,
      },
      path: {
        type: Sequelize.STRING,
        allowNull: false,
        unique: true,
      },
      created_at: {
        type: Sequelize.DATE,
        allowNull: false,
      },
      updated_at: {
        type: Sequelize.DATE,
        allowNull: false,
      },
    });
  },

  down: queryInterface => {
    return queryInterface.dropTable('files');
  },
};
```

E para gerar a tabela files no banco de dados conforme a migration, só executar no terminal:

```
yarn sequelize db:migrate  
```

Depois criar o Model File:

```
import Sequelize, { Model } from 'sequelize';

class File extends Model {
  static init(sequelize) {
    super.init(
      {
        name: Sequelize.STRING,
        path: Sequelize.STRING,
      },
      {
        sequelize,
      }
    );

    return this;
  }
}

export default File;
```

E inserir o Model File no arquivo database/index.js 


```
...
import File from  '../app/models/File';
...
const models = [User, File];
...
```

E agora atualizar o FileController.js para poder receber os dados da requisição do arquivo de upload, e salvar no banco de dados as referências:

```
import File from '../models/File';

class FileController {
  async store(req, res) {
    const { originalname: name, filename: path } = req.file;

    const file = await File.create({ name, path });

    return res.json(file);
  }
}

export default new FileController();
```

Agora na hora que enviar novamente o arquivo, a tabela dados irá ser preenchida.


### Relacionando o usuário com imagem de avatar (user <-> files)

Para fazer o relacionamento precisamsos adicionar as chaves primária de files no users.

Para isso teremos que criar uma migration para atualizar essas tabelas:

`yarn sequelize migration:create --name=add-avatar-field-to-users`

Adicionamos a coluna `avatar_id` de dentro da tabela  `users`, sendo referenciadas pela tabela `files` no atributo `ID` que é a chave primária da tabela `files`. E quando desfazer a migration é só apagar o atributo `avatar_id` de `users`.

`onUpdate: 'CASCADE'`: Quando atualiza a imagem, altera no usuário
`onDelete: 'SET NULL'`: Quando deletar o avatar deixa o avatar_id como null

```
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.addColumn('users', 'avatar_id', {
      type: Sequelize.INTEGER,
      references: { model: 'files', key: 'id' },
      onUpdate: 'CASCADE',
      onDelete: 'SET NULL',
      allowNull: true,
    });
  },

  down: queryInterface => {
    return queryInterface.removeColumn('users', 'avatar_id');
  },
};
```

Só executar: `yarn sequelize db:migrate` para executar a alteração na tabela `users`.

Depois precisa relacionar o Users com Files de dentro do Model de users no código.

Adicionando um método para associar as duas entidades:

`User.js`:
```
...
static associate(models) {
	this.belongsTo(models.File, { foreignKey: 'avatar_id' });
}
...
```

E dentro do `database/index.js`, acresento mais um `map`, para poder executar (apenas nas classes que contém o método associate) a associação e passar os models para o associate:

```
 models
      .map(model => model.init(this.connection))
      .map(model => model.associate && model.associate(this.connection.models));
```

Pronto, agora sim, na hora de salvar o usuário a associação irá ocorrer e o ID que for informado no files estará no users.


## Aula 20 - Listagem de Prestadores de Serviços

Vamos Listar todos os prestadores de servicos da aplicacao.

Como estamos falando de uma nova entidade vamos criar um novo controller `ProviderController.js`:

Dentro de `ProviderController`: 

```
import User from '../models/User';
import File from '../models/File';

class ProviderController {
  async index(req, res) {
    const providers = await User.findAll({
      where: { provider: true },
      attributes: ['id', 'name', 'email', 'avatar_id'],
      include: [
        {
          model: File,
          as: 'avatar',
          attributes: ['name', 'path', 'url'],
        },
      ],
    });
    return res.json(providers);
  }
}

export default new ProviderController();
```

Em `routes.js`:

```
...
import ProviderController from './app/controllers/ProviderController';
...
routes.get('/providers', ProviderController.index);
...
```

Agora nosso Backend nos retorna as informacoes de todos os `providers`, mas ainda nao retorna a imagem em si, para isso vamos fazer mais algumas configurações:

Em `User.js`:

Nosso codigo que estava assim:
```
...
static associate(models) {
    this.belongsTo(models.File, { foreignKey: 'avatar_id' });
  }
...
```

Vai ficar assim:
```
...
static associate(models) {
    this.belongsTo(models.File, { foreignKey: 'avatar_id', as: 'avatar' });
  }
...  
```

Dessa forma referenciamos o `avatar_id` como sendo agora `avatar`


Agora pra referenciar a imagem e conseguir fazer com que o Frontend a veja e mostre fazemos assim:

Vamos no model `File.js` e nosso arquivo vai ficar assim:

```
import Sequelize, { Model } from 'sequelize';

class File extends Model {
  static init(sequelize) {
    super.init(
      {
        name: Sequelize.STRING,
        path: Sequelize.STRING,
        url: {
          type: Sequelize.VIRTUAL,
          get() {
            return `http://localhost:3000/files/${this.path}`;
          },
        },
      },
      {
        sequelize,
      }
    );

    return this;
  }
}

export default File;
```

O que fizemos foi adicionar no `super.init` esse novo campo `url`, que vai ser VIRTUAL, ou seja nao aparece, mas esta la, e passando como metodo `get()` retornamos nossa url da aplicacao onde sera exibida a imagem. O `this.path` refere-se ao nome da imagem como foi guardado no backend.

Agora pra exibir a imagem que foi guardada no banco de dados, vamos em `app.js` na raiz da aplicacao e adicionamos o seguinte:

```
...
import path from 'path';
...
```

E dentro de `middlewares()`: 

```
...
this.app.use(
      '/files',
      express.static(path.resolve(__dirname, '..', 'tmp', 'uploads'))
    );
...
```

Fazendo isso, no insomnia ao clicar no link da imagem ela ira abrir no navegador.

## Aula 21 - Migration e model de agendamento

Toda vez que um usuario marcar um servico com algum dos prestadores, sera gerada um novo registro na tabela de agendamentos.

Criamos a tabela rodando o comando: 

```
yarn sequelize migration:create --name=create-appointments
```

A configuracao da `create-appointments`: 
```
module.exports = {
  up: (queryInterface, Sequelize) => {
    return queryInterface.createTable('appointments', {
      id: {
        type: Sequelize.INTEGER,
        allowNull: false,
        autoIncrement: true,
        primaryKey: true,
      },
      date: {
        allowNull: false,
        type: Sequelize.DATE,
      },
      user_id: {
        type: Sequelize.INTEGER,
        references: { model: 'users', key: 'id' },
        onUpdate: 'CASCADE',
        onDelete: 'SET NULL',
        allowNull: true,
      },
      provider_id: {
        type: Sequelize.INTEGER,
        references: { model: 'users', key: 'id' },
        onUpdate: 'CASCADE',
        onDelete: 'SET NULL',
        allowNull: true,
      },
      canceled_at: {
        type: Sequelize.DATE,
      },
      created_at: {
        type: Sequelize.DATE,
        allowNull: false,
      },
      updated_at: {
        type: Sequelize.DATE,
        allowNull: false,
      },
    });
  },

  down: queryInterface => {
    return queryInterface.dropTable('appointments');
  },
};

```

Fizemos ali no `user_id` a referencia do agendamento(appointments) pro usuario(user) que esta fazendo esse agendamento e a referencia desse agendamento para o provedor que vai fornecer esse
trabalho`provider_id` usando as references para relacionar duas tabelas.

Depois de configurado rodamos no terminal o comando:

```
yarn sequelize db:migrate
```

Agora criamos um `model` pra nossa tabela chamada `Appointment.js`

Dentro do model:

```
import Sequelize, { Model } from 'sequelize';

class Appointment extends Model {
  static init(sequelize) {
    super.init(
      {
        date: Sequelize.DATE,
        canceled_at: Sequelize.DATE,
      },
      {
        sequelize,
      }
    );

    return this;
  }

  static associate(models) {
    this.belongsTo(models.User, { foreignKey: 'user_id', as: 'user' });
    this.belongsTo(models.User, { foreignKey: 'provider_id', as: 'provider' });
  }
}

export default Appointment;

```

Nao precisamos referenciar o `user_id` e `provider_id` dentro do `super.init` pois como sao dados de tabelas associadas e nao referentes a tabela `appointments` nos temos que passar eles(user_id, provider_id), dentro do relacionamento `static` e somos obrigados(quando temos dois relacionamentos na mesam tabela) a passar um apelido para eles para o nosso banco de dados nao se confudir: 
``` 
as: 'user'      EX DE APELIDO
```

Depois disso vamos em ` database > index.js` e cadastramos o `appointment` como novo model: 

```
...
import Appointment from '../app/models/Appointment';
...
const models = [User, File, Appointment];
...
```

## Aula 22 - Agendamento de Servicos

Vamos criar a rota para um usuario pode agendar um servico com um prestador de servico

Criamos um `controller` chamado `AppointmentController.js`.

Dentro de `AppointmentController.js`: 

```
import * as Yup from 'yup';
import User from '../models/User';
import Appointment from '../models/Appointment';

class AppointmentController {
  async store(req, res) {
    const schema = Yup.object().shape({
      provider_id: Yup.number().required(),
      date: Yup.date().required(),
    });

    if (!(await schema.isValid(req.body))) {
      return res.status(400).json({ error: 'Validation fails' });
    }

    const { provider_id, date } = req.body;

    // check id provider_id is a provider

    const isProvider = await User.findOne({
      where: { id: provider_id, provider: true },
    });

    if (!isProvider) {
      return res
        .status(401)
        .json({ error: 'You can only create appointments with providers' });
    }

    const appointment = await Appointment.create({
      user_id: req.userId,
      provider_id,
      date,
    });

    return res.json(appointment);
  }
}

export default new AppointmentController();

```
Aqui fazemos a validacao dos dados com o Yup, verificamos se os dados estao preenchidos e corretos,
verificamos se a `id` que o usuario esta passando ao tentar fazer um agendamento é de um provider,e se nao for nos mandamos um erro `You can only create appointments with providers`. Se passar por tudo nos criamos o agendamento com o usuario que fez o agendamento, o provider que vai atender ele e a data do servico.



Depois em `routes.js`: 

```
...
import AppointmentController from './app/controllers/AppointmentController';
...
routes.post('/appointments', AppointmentController.store);
...
```

## Aula 23 - Validações de agendamento.

Vamos fazer uma validacao para verificar se a data que o usuario colocou no agendamento é uma data que esta para acontecer e nao uma data passada. A segunda validacao é pra verificar se a data de agendamento ja nao esta reservada(somente um agendamento por hora).

Vamos baixar uma biblioteca para lidar com datas dentro da nossa aplicacao:

`yarn add date-fns@next`

Dentro de `AppointmentController.js`:

```
import { startOfHour, parseISO, isBefore } from 'date-fns';

// Verificar se a data ja passou
    const hourStart = startOfHour(parseISO(date));

    if (isBefore(hourStart, new Date())) {
      return res.status(400).json({ error: 'Past dates are not permited' });
    }

    // Verificar se o provider ja nao tem algum marcado naquele horario
    const checkAvailability = await Appointment.findOne({
      where: {
        provider_id,
        canceled_at: null,
        date: hourStart,
      },
    });

    if (checkAvailability) {
      return res
        .status(400)
        .json({ error: ' Appointment date is not available' });
    }


```

E o nosso `Appointment.create` que estava assim:

```
const appointment = await Appointment.create({
      user_id: req.userId,
      provider_id,
      date,
    });
```

Vai ficar assim: 

```
const appointment = await Appointment.create({
      user_id: req.userId,
      provider_id,
      date: hourStart,
    });
```

Isso garante que nao sejam criados agendamentos em horarios quebrados.

## Aula 24 - Listando agendamento de usuários

Vamos listar todos os agendamentos que foram marcados

Dentro de `routes.js`:

`routes.get('/appointments', AppointmentController.index);`

Dentro de `AppointmentController.js`:

```
...
import File from '../models/File';
...
async index(req, res) {
    const appointments = await Appointment.findAll({
      where: { user_id: req.userId, canceled_at: null },
      order: ['date'],
      attributes: ['id', 'date'],
      include: [
        {
          model: User,
          as: 'provider',
          attributes: ['id', 'name'],
          include: [
            {
              model: File,
              as: 'avatar',
              attributes: ['id', 'path', 'url'],
            },
          ],
        },
      ],
    });
    return res.json(appointments);
  }
  ...
```

Para entender o que estamos fazendo:

Fizemos a chamada de um metodo `index` que como aprendemos no inicio do curso é para listagem de tudo que esta dentro de uma tabela, e pegamos esse dados baseados na `id` e se o usuario nao cancelou o agendamento, ali no `order['date']` estamos dizendo para serem listados do mais recente para o mais antigo, e somente exibir os dados `id` e `date`  do appointment, depois disso incluimos os dados tais como foto do `provider` fazendo referencia ao model `User`.

## Aula 25 - Aplicando paginaçāo

Quando o usuario for carregar os agendamento no Frontend e tiver 200 agendamentos no banco de dados nao é legal aparecerem os 200, entao vamos dividir em paginas que vao conter 20 agendamentos cada.

La no `AppointmentController.js`: 

```
async index(req, res) {
  * const { page = 1 } = req.query;

    const appointments = await Appointment.findAll({
      where: { user_id: req.userId, canceled_at: null },
      order: ['date'],
      attributes: ['id', 'date'],
    * limit: 20,
    * offset: (page - 1) * 20,
      include: [
        {
          model: User,
          as: 'provider',
          attributes: ['id', 'name'],
          include: [
            {
              model: File,
              as: 'avatar',
              attributes: ['id', 'path', 'url'],
            },
          ],
        },
      ],
    });
    return res.json(appointments);
  }
```
Obs: tire os asteriscos do codigo.

A conta é bem simples, a `const page` por padrao comeca em 1, ali dizemos para o`offset` pegar o valor de `page` diminuir de 1 e listar os proximos 20.

ex: se page for 1, ele vai diminuir de 1 e multiplicar por 20 e exibir os proximos 20 resultados(1 - 1) * 20 = 0, ou seja, nao vai pular nada e vai exibir os 20 primeiros, se page fosse 2, ele ia fazer a conta e multiplicar e ia dar 20, ou seja, ia pular 20 agendamentos e exibir os proximos 20.

## Aula 26 - Listando agenda do prestador

Quando o determinado prestador de servicos entrar na aplicacao ele vai ver um painel com a listagem de todos os agendamentos que ele tem no dia, vai ser uma listagem unica pra cada provider.

Criamos um novo controller `ScheduleController.js`:

```
import { startOfDay, endOfDay, parseISO } from 'date-fns';
import { Op } from 'sequelize';

import User from '../models/User';
import Appointment from '../models/Appointment';

class ScheduleController {
  async index(req, res) {
    const checkUserProvider = await User.findOne({
      where: { id: req.userId, provider: true },
    });

    if (!checkUserProvider) {
      return res.status(401).json({ error: 'User is not a provider' });
    }

    const { date } = req.query;
    const parsedDate = parseISO(date);

    const appointments = await Appointment.findAll({
      where: {
        provider_id: req.userId,
        canceled_at: null,
        date: {
          [Op.between]: [startOfDay(parsedDate), endOfDay(parsedDate)],
        },
      },
      order: ['date'],
    });

    return res.json(appointments);
  }
}

export default new ScheduleController();
```
Vamos primeiro verificar se o usuario é um provider, depois vamos listar todos os agendamentos daquele prestador de servico, que nao estejam cancelados, e que estejam entre o inicio do dia (00:00:00) e o final do dia(23:59:59) e vamos ordenar esses agendamentos por data.


Em `routes.js`: 

```
...
import ScheduleController from './app/controllers/ScheduleController';
...
routes.get('/schedule', ScheduleController.index);
...
```

## Aula 27 - Configurando MongoDB

Vamos conectar a outro banco de dados (já estamos conectados ao PostgresSQL) pois vamos ter dados que nao vao ser relacionados dentro da nossa aplicacao e por isso vamos usar um banco de dados nao relacional (NoSQL) chamado MongoDB.

Para iniciar rode o comando na pasta do projeto: 

`docker run --name mongobarber -p 27017:27017 -d -t mongo `

Se aparecer um codigo tipo assim é pq funcionou:

`22576dfaecd1c24a9128db110b0a0235f7243b0ee73f4ae0b8ffc2646287f3f3`

Para ver se o mongo esta rodando vamos no navegador e digitamos `localhost:27017` e se aparecer essa mensagem é porque o mongo ja esta rodando:

`It looks like you are trying to access MongoDB over HTTP on the native driver port.`

### - Conectando a aplicaçāo com o MongoDB:

Assim como instalamos o `Sequelize` para o PostgresSQL vamos intalar o `Mongoose` para lidar com o MongoDB:

`yarn add mongoose`

Agora pra configura-lo vamos no nosso arquivo ja existente em `src > database > index.js` e o arquivo que estava assim:

```
import Sequelize from 'sequelize';

import User from '../app/models/User';
import File from '../app/models/File';
import Appointment from '../app/models/Appointment';

import databaseConfig from '../config/database';

const models = [User, File, Appointment];

class Database {
  constructor() {
    this.init();
  }

  init() {
    this.connection = new Sequelize(databaseConfig);

    models
      .map(model => model.init(this.connection))
      .map(model => model.associate && model.associate(this.connection.models));
  }
}

export default new Database();

```

vai ficar assim (tire os *):

```
import Sequelize from 'sequelize';
* import mongoose from 'mongoose';

import User from '../app/models/User';
import File from '../app/models/File';
import Appointment from '../app/models/Appointment';

import databaseConfig from '../config/database';

const models = [User, File, Appointment];

class Database {
  constructor() {
    this.init();
  * this.mongo();
  }

  init() {
    this.connection = new Sequelize(databaseConfig);

    models
      .map(model => model.init(this.connection))
      .map(model => model.associate && model.associate(this.connection.models));
  }

 * mongo() {
 *   this.mongoConnection = mongoose.connect(
 *     'mongodb://localhost:27017/gobarber',
 *     {
 *       useNewUrlParser: true,
 *       useFindAndModify: true,
 *       useUnifiedTopology: true,
 *     }
 *   );
 * }
}

export default new Database();

```
Se rodarmos a aplicacao `yarn dev` ou `yarn nodemon` no meu caso e nao der erro é porque foi configurado certo e ja esta funcionando.

## Aula 28 - Notificando novos agendamentos

Vamos enviar um notificacao pro prestador de servicos toda vez que ele receber um novo agendamento, e vamos utilizar o Mongo para armazenar essas notificacoes.

Dentro da pasta `app` vamos criar um nova pasta chamada `schemas` e dentro dela vamos criar um arquivo chamado `Notification.js`.

Dentro de `Notification.js`:

```
import mongoose from 'mongoose';

const NotificationSchema = new mongoose.Schema(
  {
    content: {
      type: String,
      required: true,
    },
    user: {
      type: Number,
      required: true,
    },
    read: {
      type: Boolean,
      required: true,
    },
  },
  {
    timestamps: true,
  }
);

export default mongoose.model('Notification', NotificationSchema);
```
A vantagem do NoSQL é que nao precisamos fazer uma `migration` para cada `model`, e podemos tambem importar diretamente a `model` no arquivo que queremos e já sair utilizando.

Depois disso vamos em `AppointmentController.js` pois é onde gerenciamos os agendamentos.

Em `AppointmentController.js`(tire os *):

```
...
  import { startOfHour, parseISO, isBefore, ** format ** } from 'date-fns';
  import pt from 'date-fns/locale/pt';
...
  import Notification from '../schemas/Notification';
...

```


ainda em `AppointmentController.js` dentro do metodo `store()` logo após criar o `appointment`:

```
const user = await User.findByPk(req.userId);
    const formattedDate = format(
      hourStart,
      "'dia' dd 'de' MMMM', às' H:mm'h'",
      { locale: pt }
    );

    await Notification.create({
      content: `Novo agendamento de ${user.name} para ${formattedDate}`,
      user: provider_id,
    });

```

Agora se criarmos um agendamento(appointment), ele vai armazenar no banco de dados a notificacao para futuramente enviarmos em notificacao para o prestador de servico(provider).


## Aula 29 - Listando Notificações do Usuário

Vamos criar uma rota que lista as notificações do prestador de serviço.

Vamos em `routes.js`:

```
...
import NotificationController from './app/controllers/NotificationController';
...
routes.get('/notifications', NotificationController.index);
...
```

depois vamos la na pasta `controllers` e criamos um arquivo `NotificationController.js`

dentro de `NotificationController.js`:

```
import User from '../models/User';
import Notification from '../schemas/Notification';

class NotificationController {
  async index(req, res) {
    const isProvider = await User.findOne({
      where: { id: req.userId, provider: true },
    });
    if (!isProvider) {
      return res
        .status(401)
        .json({ error: 'Only providers can view notifications' });
    }

    const notifications = await Notification.find({
      user: req.userId,
    })
      .sort({ createdAt: 'desc' })
      .limit(20);

    return res.json(notifications);
  }
}

export default new NotificationController();
```
Primeiro estamos verificando se a pessoa que esta tentando lista é um prestador de servicos, depois a gente pega somente as notificacoes que tem pro usuario que esta tentando listar(nao queremos que listem todos os agendamento de todos os prestadores) e exibindo no final.

## Aula 30 - Marcar notificações como lidas

Dentro de `routes.js`:

`routes.put('/notifications/:id', NotificationController.update);`

Em `NotificationController` vamos criar um novo metodo `update`:

```
async update(req, res) {
    const notification = await Notification.findByIdAndUpdate(
      req.params.id,
      { read: true },
      { new: true }
    );
    return res.json(notification);
  }
```

Passamos a id da notificacao, depois passamos o que ira atualizar `{ read: true }` quando chamarmos aquela rota, pois a pessoa nao escolhe se vai vizualizar, a partir do momento que ela entra ja muda o `read` para `true` e depois passamos esse esse `{ new: true }` que ira dizer para o mongo retornar, pois se nao ele ira atualizar no banco de dados mas nao retornara o valor