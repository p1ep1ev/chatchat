<template>
  <div class="container">
    <h1 class="text-center">Welcome to Chatire!</h1>
    <div id="auth-container" class="row">
      <div class="col-sm-4 offset-sm-4">
        <ul class="nav nav-tabs nav-justified" id="myTab" role="tablist">
          <li class="nav-item">
            <a class="nav-link" :class="{active: authStep === 'signup'}" @click="authStep = 'signup'" href="#">Sign Up</a>
          </li>
          <li class="nav-item">
            <a class="nav-link" :class="{active: authStep === 'signin'}" @click="authStep = 'signin'" href="#">Sign In</a>
          </li>
        </ul>

        <div class="tab-content" id="myTabContent">

          <div v-if="authStep === 'signup'" class="tab-pane show active" id="signup" role="tabpanel" aria-labelledby="signin-tab">
            <form @submit.prevent="signUp">
              <div class="form-group">
                <input v-model="email" type="email" class="form-control" id="email" placeholder="Email Address" required>
              </div>
              <div class="form-row">
                <div class="form-group col-md-6">
                  <input v-model="username" type="text" class="form-control" id="username" placeholder="Username" required>
                </div>
                <div class="form-group col-md-6">
                  <input v-model="password" type="password" class="form-control" id="password" placeholder="Password" required>
                </div>
              </div>
              <button type="submit" class="btn btn-block btn-primary">Sign up</button>
            </form>
          </div>

          <div v-if="authStep === 'signin'" class="tab-pane show active">
            <form @submit.prevent="signIn">
              <div class="form-group">
                <input v-model="username" type="text" class="form-control" id="username" placeholder="Username" required>
              </div>
              <div class="form-group">
                <input v-model="password" type="password" class="form-control" id="password" placeholder="Password" required>
              </div>
              <button type="submit" class="btn btn-block btn-primary">Sign in</button>
            </form>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  data () {
    return {
      email: '', username: '', password: '', authStep: 'signup'
    }
  },
  methods: {
    signUp () {
      const payload = {
        email: this.email, // wow email
        username: this.username,
        password: this.password
      }
      this.$.post('http://localhost:8000/auth/users/', payload, (data) => {
        alert("Your account has been created. You will be signed in automatically")
        this.signIn()
      })
      .fail((response) => {
        alert("Ошибка регистрации: " + JSON.stringify(response.responseJSON))
      })
    },
    signIn () {
      const credentials = {username: this.username, password: this.password}

    this.$.post('http://localhost:8000/auth/token/login/', credentials, (data) => {
        sessionStorage.setItem('authToken', data.auth_token)
        sessionStorage.setItem('username', this.username)
        this.$router.push('/chats')
      })
      .fail((response) => {
        alert("Ошибка входа: проверьте логин и пароль")
      })
    }
  }
}
</script>

<style scoped>
  #auth-container {
    margin-top: 50px;
  }

  .tab-content {
    padding-top: 20px;
  }
</style>
