
# Continuando API do GoBarber

## Aula 18 - Configurando Multer

### Upload de imagem

Usuário seleciona a imagem, o upload já é feito e o servidor retorna o ID da imagem.
E no json no cadastro de usuário por exemplo, envia o ID da imagem.

Utilizando o [multer](https://github.com/expressjs/multer) para upload de arquivos.

Quando precisa enviar imagem para o servidor, tem que ser como `Multpart-data` (Multpart Form) e não `json`.

Instalando o `multer`: 

```
yarn add multer
```

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

```
yarn sequelize migration:create --name=add-avatar-field-to-users
```

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

`Users.js`:
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

Vamos baixar uma biblioteca para lidar co datas dentro da nossa aplicacao:

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









