<template>
	<el-container class="register-view">
		<div>
			
			<el-form :model="registerForm" status-icon :rules="rules" ref="registerForm" label-width="80px" class="web-ruleForm">
				<div class="register-brand">欢迎注册</div>
				<el-form-item label="用户名" prop="userName">
					<el-input type="userName" v-model="registerForm.userName" placeholder="登录账号名称" autocomplete="off"></el-input>
				</el-form-item>
				<el-form-item label="昵称" prop="nickName">
					<el-input type="nickName" v-model="registerForm.nickName" placeholder="用户昵称" autocomplete="off"></el-input>
				</el-form-item>
				<el-form-item label="密码" prop="password">
					<el-input type="password" v-model="registerForm.password" placeholder="请输入密码" autocomplete="off"></el-input>
				</el-form-item>
				<el-form-item label="确认密码" prop="confirmPassword">
					<el-input type="password" v-model="registerForm.confirmPassword" placeholder="再次输入密码" autocomplete="off"></el-input>
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="submitForm('registerForm')">注册</el-button>
					<el-button @click="resetForm('registerForm')">清空</el-button>
				</el-form-item>
				<div class="to-login">
					<router-link to="/login">已有账号,前往登录</router-link>
				</div>
			</el-form>
		</div>
	</el-container>
</template>

<script>
	export default {
		name: "login",
		data() {
			var checkUserName = (rule, value, callback) => {
				if (!value) {
					return callback(new Error('请输入用户名'));
				}
				callback();
			};
			var checkNickName = (rule, value, callback) => {
				if (!value) {
					return callback(new Error('请输入昵称'));
				}
				callback();
			};
			var checkPassword = (rule, value, callback) => {
				if (value === '') {
					return callback(new Error('请输入密码'));
				}
				callback();
			};

			var checkConfirmPassword = (rule, value, callback) => {
				console.log("checkConfirmPassword");
				if (value === '') {
					return callback(new Error('请输入密码'));
				}
				if (value != this.registerForm.password) {
					return callback(new Error('两次密码输入不一致'));
				}
				callback();
			};


			return {
				registerForm: {
					userName: '',
					nickName: '',
					password: '',
					confirmPassword: ''
				},
				rules: {
					userName: [{
						validator: checkUserName,
						trigger: 'blur'
					}],
					nickName: [{
						validator: checkNickName,
						trigger: 'blur'
					}],
					password: [{
						validator: checkPassword,
						trigger: 'blur'
					}],
					confirmPassword: [{
						validator: checkConfirmPassword,
						trigger: 'blur'
					}]
				}
			};
		},
		methods: {
			submitForm(formName) {
				this.$refs[formName].validate((valid) => {
					if (valid) {
						this.$http({
								url: "/register",
								method: 'post',
								data: this.registerForm
							})
							.then((data) => {
								this.$message.success("注册成功!");
                this.$router.push("/login");
							}).catch((err) => {
                this.$message.error("服务异常!",err);
            });
					}
				});
			},
			resetForm(formName) {
				this.$refs[formName].resetFields();
			}
		}
	}
</script>

<style scoped lang="scss">
	.register-view {
		position: fixed;
		display: flex;
		justify-content: space-around;
		width: 100%;
		height: 100%;
		background: #466368;
    background-image: url(~@/assets/images/b3.jpg);
		background-size: cover;
		-webkit-user-select: none;
		background-size: cover;
		
		
		.web-ruleForm {
			width: 500px;
			height: 430px;
			padding: 20px;
			margin-top: 100px ;
			background: rgba(255,255,255,.75);
			box-shadow: 0px 0px  1px #ccc;
			border-radius: 3px;
			overflow: hidden;
			
			.register-brand {
				line-height: 50px;
				margin: 20px 0 30px 0;
				font-size: 22px;
				font-weight: 600;
				letter-spacing: 2px;
				text-align: center;
				text-transform: uppercase;
			}
			
			.to-login {
				display: flex;
				flex-direction: row-reverse;
				line-height: 40px;
				text-align: left;
				padding-left: 20px;
			}
		}
	}

	

	
</style>
