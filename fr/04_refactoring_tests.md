# Factoriser les tests {#chapter:4}

Dans le chapitre [6](#chapter:3){reference-type="ref"reference="chapter:3"} nous avons mis en place des entrée de ressources utilisateur. Si vous avez sauté ce chapitre ou si vous n'avez pas tout compris, je vous recommande vivement de le regarder. Il couvre les premières bases des tests et c'est une introduction aux réponses JSON.

Vous pouvez cloner le projet à cette étape avec:

~~~bash
$ git clone --branch chapitre_3 https://github.com/madeindjs/market_place_api
~~~

Dans ce chapitre, nous allons factoriser nos spécifications de test en ajoutant des méthodes d'aide, supprimer le paramètre de format envoyé sur chaque requête et le faire à travers les en-têtes, et nous espérons construire une suite de test plus cohérente et évolutive.

Jetons donc un coup d'oeil au fichier `users_controller_spec.rb` (Listing [\[lst:users\_controller\_spec\_before\_factorization\]](#lst:users_controller_spec_before_factorization){reference-type="ref"reference="lst:users_controller_spec_before_factorization"}):

~~~ruby
# spec/controllers/api/v1/users_controller_spec.rb

# ...

RSpec.describe Api::V1::UsersController, type: :controller do
  before(:each) { request.headers['Accept'] = 'application/vnd.marketplace.v1' }

  describe 'GET #show' do
    before(:each) do
      @user = FactoryBot.create :user
      get :show, params: { id: @user.id, format: :json }
    end

    it 'returns the information about a reporter on a hash' do
      user_response = JSON.parse(response.body, symbolize_names: true)
      expect(user_response[:email]).to eql @user.email
    end

    it { expect(response.response_code).to eq(200) }
  end

  describe 'POST #create' do
    context 'when is successfully created' do
      before(:each) do
        @user_attributes = FactoryBot.attributes_for :user
        post :create, params: { user: @user_attributes }, format: :json
      end

      it 'renders the json representation for the user record just created' do
        user_response = JSON.parse(response.body, symbolize_names: true)
        expect(user_response[:email]).to eql @user_attributes[:email]
      end

      it { expect(response.response_code).to eq(201) }
    end

    context 'when is not created' do
      before(:each) do
        # notice I'm not including the email
        @invalid_user_attributes = { password: '12345678',
                                     password_confirmation: '12345678' }
        post :create, params: { user: @invalid_user_attributes }, format: :json
      end

      it 'renders an errors json' do
        user_response = JSON.parse(response.body, symbolize_names: true)
        expect(user_response).to have_key(:errors)
      end

      it 'renders the json errors on why the user could not be created' do
        user_response = JSON.parse(response.body, symbolize_names: true)
        expect(user_response[:errors][:email]).to include "can't be blank"
      end

      it {  expect(response.response_code).to eq(422) }
    end
  end


  describe "PUT/PATCH #update" do

   context "when is successfully updated" do
     before(:each) do
       @user = FactoryBot.create :user
       patch :update, params: {
         id: @user.id,
         user: { email: "newmail@example.com" } },
         format: :json
     end

     it "renders the json representation for the updated user" do
       user_response = JSON.parse(response.body, symbolize_names: true)
       expect(user_response[:email]).to eql "newmail@example.com"
     end

     it {  expect(response.response_code).to eq(200) }
   end

   context "when is not created" do
     before(:each) do
       @user = FactoryBot.create :user
       patch :update, params: {
         id: @user.id,
         user: { email: "bademail.com" } },
         format: :json
     end

     it "renders an errors json" do
       user_response = JSON.parse(response.body, symbolize_names: true)
       expect(user_response).to have_key(:errors)
     end

     it "renders the json errors on whye the user could not be created" do
       user_response = JSON.parse(response.body, symbolize_names: true)
       expect(user_response[:errors][:email]).to include "is invalid"
     end

     it {  expect(response.response_code).to eq(422) }
   end
  end

  describe "DELETE #destroy" do
    before(:each) do
      @user = FactoryBot.create :user
      delete :destroy, params: { id: @user.id }, format: :json
    end

    it { expect(response.response_code).to eq(204) }
  end
end
~~~

Comme vous pouvez le voir, il y a beaucoup de code dupliqué. Deux possibilité de factorisation sont:

- La méthode `JSON.parse` peut être encapsulée sur une méthode.
- Le paramètre format est envoyé à chaque demande. Bien que ce ne soit pas une mauvaise pratique, il est préférable de gérer le type de réponse à l'aide des en-têtes.

Ajoutons donc une méthode pour gérer la réponse JSON. Mais avant de
continuer, et si vous avez suivi le tutoriel, vous savez peut-être que
nous créons une branche pour chaque chapitre. Alors faisons-le:

~~~bash
$ git checkout -b chapter4
~~~

## Refactorisation de la réponse JSON

De retour à notre factorisation, nous allons créer un fichier sous le répertoire `spec/support`. Actuellement, nous n'avons pas ce répertoire, alors créons-le:

~~~bash
$ mkdir spec/support
~~~

Ensuite, nous créons un fichier `request_helpers.rb` sous le répertoire `support` que nous venons de créer:

~~~bash
$ touch spec/support/request_helpers.rb
~~~

Il est temps d'extraire la méthode `JSON.parse` dans notre propre méthode de support (Listing [\[lst:create\_request\_helpers\]](#lst:create_request_helpers){reference-type="ref"reference="lst:create_request_helpers"}):

~~~ruby
# spec/support/request_helpers.rb
module Request
  module JsonHelpers
    def json_response
      @json_response ||= JSON.parse(response.body, symbolize_names: true)
    end
  end
end
~~~

Nous allons intégrer la méthode dans certains `modules` afin de garder notre code bien organisé. L'étape suivante consiste à mettre à jour le fichier `users_controller_spec.rb` pour utiliser la méthode. Un exemple rapide est présenté ci-dessous:

~~~ruby
# spec/controllers/api/v1/users_controller_spec.rb

# ...
it 'returns the information about a reporter on a hash' do
  user_response = json_response # c'est cette ligne qui est maj
  expect(user_response[:email]).to eql @user.email
end
# ...
~~~

C'est maintenant à votre tour de mettre à jour l'ensemble du fichier!

Si vous essayez maintenant d'exécuter vos tests avec `bundle exec rspec spec/controllers` vous allez avoir une erreur. C'est normal. La méthode `json_response` n'est pas chargée dans le fichier `rails_helper.rb`. Il faut donc modifier un peu notre `rails_helper` qui s'occupe de configurer nos tests:

~~~ruby
# spec/rails_helper.rb

# chargement de tous les fichiers Ruby dans le dossier spec/support
Dir[Rails.root.join('spec', 'support', '**', '*.rb')].each do |f|
  require f
end

RSpec.configure do |config|
  #  ...
  # Nous devons aussi inclure ces methodes dans rspec en tant
  # qu'aides de type controleur
  config.include Request::JsonHelpers, :type => :controller
  #  ...
end
~~~

Une fois le fichier modifié, nos tests devraient passer à nouveau! *Commitons* donc ceci avant d'aller plus loin:

~~~bash
$ git add .
$ git commit -m "Refactors the json parse method"
~~~

## Factoriser le paramètre du format

Nous voulons supprimer les paramètres `format: :json` envoyé sur chaque requête. Pour le faire c'est extrêmement facile. Il suffit simplement d'ajouter une ligne à notre fichier `users_controller_spec.rb`:

~~~ruby
# spec/controllers/api/v1/users_controller_spec.rb

RSpec.describe Api::V1::UsersController, type: :controller do
  before(:each) { request.headers['Accept'] = "application/vnd.marketplace.v1, application/json" }
~~~

En ajoutant cette ligne, vous pouvez maintenant supprimer tous les paramètres de `format` que nous envoyions sur chaque requête!

Attendez, ce n'est pas encore fini! Nous pouvons ajouter un autre en-tête à notre demande qui nous aidera à décrire les données que nous attendons du serveur à livrer. Nous pouvons y parvenir assez facilement en ajoutant une ligne supplémentaire spécifiant l'en-tête `Content-Type`:

~~~ruby
# spec/controllers/api/v1/users_controller_spec.rb

RSpec.describe Api::V1::UsersController, type: :controller do
  before(:each) { request.headers['Accept'] = "application/vnd.marketplace.v1, application/json" }
  before(:each) { request.headers['Content-Type'] = 'application/json' }
~~~

Et encore une fois,nous lançons nos tests pour voir si tout est bon:

~~~bash
$ bundle exec rspec spec/controllers/api/v1/users_controller_spec.rb
.............

Finished in 1.44 seconds (files took 0.4734 seconds to load)
13 examples, 0 failures
~~~

Et comme à chaque fois, c'est le bon moment pour `commit`:

~~~bash
$ git commit -am "Factorize format for unit tests"
~~~

## Factoriser le paramètre du format

Je suis vraiment satisfait du code que nous avons obtenu, mais nous pouvons faire encore mieux. La première chose qui me vient à l'esprit est de regrouper les 3 en-têtes personnalisés ajoutés avant chaque requête:

~~~ruby
# spec/controllers/api/v1/users_controller_spec.rb
#...

before(:each) { request.headers['Accept'] = "application/vnd.marketplace.v1, application/json" }
before(:each) { request.headers['Content-Type'] = 'application/json' }
~~~

C'est bien mais on peut mieux faire. En effet, nous devrons ajouter ces cinq lignes de code pour chaque fichier. Si pour une raison quelconque, nous changeons le type de réponse en XML, nous devrions modifier les cinq fichiers manuellement. Ne vous inquiétez pas, je vais vous proposer une solution qui résoudra tous ces problèmes.

Tout d'abord, nous devons étendre notre fichier `request_helpers.rb` pour inclure un autre module que j'ai nommé `HeadersHelpers` et qui aura les méthodes nécessaires pour gérer ces en-têtes personnalisés (Listing [\[lst:add\_headers\_to\_request\_helpers\]](#lst:add_headers_to_request_helpers){reference-type="ref"reference="lst:add_headers_to_request_helpers"})

~~~ruby
# spec/support/request_helpers.rb

module Request
  # ...

  module HeadersHelpers
    def api_header(version = 1)
      request.headers['Accept'] = "application/vnd.marketplace.v#{version}"
    end

    def api_response_format(format ='application/json')
      request.headers['Accept'] = "#{request.headers['Accept']}, #{format}"
      request.headers['Content-Type'] = format
    end

    def include_default_accept_headers
      api_header
      api_response_format
    end
  end
end
~~~

Comme vous pouvez le voir, j'ai divisé les appels en deux méthodes: une pour définir l'en-tête API et l'autre pour définir le format de réponse. J'ai aussi écrit une méthode (`include_default_accept_headers`) pour appeler les deux.

Et maintenant, pour appeler cette méthode avant chacun de nos test, nous pouvons ajouter le `before` dans le bloc `Rspec.configure` du fichier `rails_helper.rb` (Listing [\[lst:complete\_spec\_rails\_helper\]](#lst:complete_spec_rails_helper){reference-type="ref"reference="lst:complete_spec_rails_helper"}), et nous assurer de spécifier le type au `:controller` car nous ne le faisons que pour les tests unitaires concernant les controlleurs.

~~~ruby
# spec/rails_helper.rb

# ...

RSpec.configure do |config|
  # ...
  config.include Request::HeadersHelpers, :type => :controller
  config.before(:each, type: :controller) do
    include_default_accept_headers
  end

  # ...
end
~~~

Après avoir ajouté ces lignes, nous pouvons supprimer les `before` avant sur le fichier `users_controller_spec.rb` et vérifier que nos tests passent toujours.

Vous pouvez consulter la version complète du fichier `spec_helper.rb` ci-dessous (Listing [\[lst:complete\_spec\_rails\_helper\]](#lst:complete_spec_rails_helper){reference-type="ref"reference="lst:complete_spec_rails_helper"}):

~~~ruby
# spec/rails_helper.rb

require 'spec_helper'
ENV['RAILS_ENV'] ||= 'test'
require File.expand_path('../../config/environment', __FILE__)
# Prevent database truncation if the environment is production
abort("The Rails environment is running in production mode!") if Rails.env.production?
require 'rspec/rails'

Dir[Rails.root.join('spec', 'support', '**', '*.rb')].each { |f| require f }

begin
  ActiveRecord::Migration.maintain_test_schema!
rescue ActiveRecord::PendingMigrationError => e
  puts e.to_s.strip
  exit 1
end

RSpec.configure do |config|
  config.fixture_path = "#{::Rails.root}/spec/fixtures"

  config.use_transactional_fixtures = true

  config.include Devise::Test::ControllerHelpers, type: :controller
  config.include Request::JsonHelpers, :type => :controller
  config.include Request::HeadersHelpers, :type => :controller
  config.before(:each, type: :controller) do
    include_default_accept_headers
  end


  config.infer_spec_type_from_file_location!

  config.filter_rails_from_backtrace!
end
~~~

Et bien maintenant je suis satisfait du code. *Commitons* nos changements:

~~~bash
$ git commit -am "Refactors test headers for each request"
~~~

Rappelez-vous que vous pouvez revoir le code jusqu'à ce point dans le [dépôt github](https://github.com/madeindjs/market_place_api).

## Conclusion

Pour finir ce chapitre, bien qu'il ait été court, c'était une étape cruciale car cela nous aidera à écrire des tests plus rapides. Au chapitre [8](#chapter:5){reference-type="ref" reference="chapter:5"}, nous ajouterons le mécanisme d'authentification que nous utiliserons à travers l'application ainsi que la restriction de l'accès à certaines actions.