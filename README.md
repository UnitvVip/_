
<html lang="pt">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>UNITV VIP</title>

<script src="https://cdn.tailwindcss.com"></script>
<script src="https://unpkg.com/lucide@latest"></script>

<style>
.hide-scrollbar::-webkit-scrollbar{display:none}
.hide-scrollbar{-ms-overflow-style:none;scrollbar-width:none}

@keyframes fadeIn{
from{opacity:0;transform:translate(-50%,-10px)}
to{opacity:1;transform:translate(-50%,0)}
}
.animate-fade-in{animation:fadeIn .3s ease-out forwards}
</style>
</head>

<body class="bg-gray-50 font-sans text-gray-800">

<div id="toast-container" class="fixed top-4 left-1/2 -translate-x-1/2 z-[100] w-max"></div>
<div id="app-root"></div>

<script>

const state={
currentView:'login',
currentUser:null,
isLoginMode:true,

users:[],

plans:[
{id:'annual',title:'Anual: 365 dias',price:249.90,oldPrice:299,monthlyAvg:20.54,stock:10},
{id:'monthly',title:'Mensal',price:39.90,oldPrice:49,monthlyAvg:39.90,stock:2},
{id:'quarterly',title:'Trimestral',price:99.90,oldPrice:139,monthlyAvg:33.30,stock:5}
],

coupons:[
{code:'VIP10',discount:10},
{code:'OFERTA5',discount:5}
],

selectedPlanId:'monthly',
appliedCoupon:null
}









/* ============================
BANCO DE DADOS LOCAL
============================ */

function saveDatabase(){

localStorage.setItem("unitv_database",
JSON.stringify({
users:state.users,
plans:state.plans,
coupons:state.coupons
})
)

}

function loadDatabase(){

const data=localStorage.getItem("unitv_database")

if(!data) return

const db=JSON.parse(data)

if(db.users)state.users=db.users
if(db.plans)state.plans=db.plans
if(db.coupons)state.coupons=db.coupons

}









/* ============================
TOAST
============================ */

function showToast(message,type='success'){

const container=document.getElementById('toast-container')

const bg=type==='error'?'bg-red-500':'bg-gray-800'
const icon=type==='error'?'alert-circle':'check-circle-2'

const toast=document.createElement('div')

toast.className=`px-6 py-3 rounded-full shadow-lg text-white text-sm flex items-center gap-2 mb-2 animate-fade-in ${bg}`

toast.innerHTML=`<i data-lucide="${icon}" class="w-4 h-4"></i>${message}`

container.appendChild(toast)

lucide.createIcons()

setTimeout(()=>toast.remove(),3000)

}









/* ============================
LOGIN / REGISTRO
============================ */

function toggleLoginMode(){
state.isLoginMode=!state.isLoginMode
render()
}

function handleAuth(e){

e.preventDefault()

const id=document.getElementById("auth-id").value.trim()
const pass=document.getElementById("auth-pass").value.trim()

if(!id||!pass){
showToast("Preencha ID e senha","error")
return
}

if(state.isLoginMode){

if(id==="admin"&&pass==="admin"){
state.currentUser={id:"Admin",role:"admin"}
state.currentView="admin"
render()
return
}

const user=state.users.find(u=>u.id===id&&u.password===pass)

if(user){

state.currentUser={id:id,role:"client"}
state.currentView="client"

render()

}else{
showToast("ID ou senha incorretos","error")
}

}else{

if(id==="admin"){
showToast("ID reservado","error")
return
}

if(state.users.some(u=>u.id===id)){
showToast("ID já existe","error")
return
}

state.users.push({id:id,password:pass})

saveDatabase()

showToast("Conta criada")

state.isLoginMode=true

render()

}

}

function logout(){
state.currentUser=null
state.currentView='login'
render()
}









/* ============================
ADMIN
============================ */

function savePlans(){

state.plans.forEach(plan=>{

const price=parseFloat(document.getElementById(`price-${plan.id}`).value)
const oldPrice=parseFloat(document.getElementById(`oldprice-${plan.id}`).value)
const stock=parseInt(document.getElementById(`stock-${plan.id}`).value)

if(!isNaN(price))plan.price=price
if(!isNaN(oldPrice))plan.oldPrice=oldPrice
if(!isNaN(stock))plan.stock=stock

})

saveDatabase()

showToast("Planos salvos")

render()

}

function addCoupon(){

const code=document.getElementById("new-coupon-code").value.trim().toUpperCase()
const discount=parseFloat(document.getElementById("new-coupon-discount").value)

if(!code||isNaN(discount)){
showToast("Dados inválidos","error")
return
}

state.coupons.push({code,discount})

saveDatabase()

render()

}

function removeCoupon(code){

state.coupons=state.coupons.filter(c=>c.code!==code)

saveDatabase()

render()

}









/* ============================
CLIENTE
============================ */

function selectPlan(id){

const plan=state.plans.find(p=>p.id===id)

if(plan.stock>0){

state.selectedPlanId=id

render()

}

}

function applyCoupon(){

const code=document.getElementById("client-coupon-input").value.trim().toUpperCase()

const coupon=state.coupons.find(c=>c.code===code)

if(!coupon){

showToast("Cupom inválido","error")

return

}

state.appliedCoupon=coupon

render()

}

function removeAppliedCoupon(){

state.appliedCoupon=null

render()

}









function handleWhatsAppRedirect(){

const phone="5598984533013"

const plan=state.plans.find(p=>p.id===state.selectedPlanId)

let price=plan.price

if(state.appliedCoupon)
price=Math.max(0,price-state.appliedCoupon.discount)

const msg=`Olá! Meu ID é ${state.currentUser.id}
Plano: ${plan.title}
Valor: R$ ${price.toFixed(2)}`

window.open(`https://wa.me/${phone}?text=${encodeURIComponent(msg)}`,'_blank')

}









/* ============================
RENDER LOGIN
============================ */

function renderLogin(){

return`

<div class="min-h-screen flex items-center justify-center">

<form onsubmit="handleAuth(event)" class="bg-white p-8 rounded-xl shadow w-80 space-y-4">

<h1 class="text-xl font-bold text-center">${state.isLoginMode?'Login':'Criar Conta'}</h1>

<input id="auth-id" placeholder="ID" class="w-full border p-2 rounded">

<input id="auth-pass" type="password" placeholder="Senha" class="w-full border p-2 rounded">

<button class="w-full bg-green-500 text-white p-2 rounded">

${state.isLoginMode?'Entrar':'Criar'}

</button>

<button type="button" onclick="toggleLoginMode()" class="text-sm text-gray-500 w-full">

${state.isLoginMode?'Criar conta':'Voltar login'}

</button>

</form>

</div>

`

}









/* ============================
RENDER ADMIN
============================ */

function renderAdmin(){

const users=state.users.map(u=>`
<div class="border p-2 rounded flex justify-between">
<span>${u.id}</span>
<span class="text-gray-400 text-xs">${u.password}</span>
</div>
`).join("")

const plans=state.plans.map(p=>`

<div class="border p-3 rounded space-y-1">

<b>${p.title}</b>

<input id="price-${p.id}" type="number" value="${p.price}" class="border p-1 w-full">

<input id="oldprice-${p.id}" type="number" value="${p.oldPrice}" class="border p-1 w-full">

<input id="stock-${p.id}" type="number" value="${p.stock}" class="border p-1 w-full">

</div>

`).join("")

const coupons=state.coupons.map(c=>`

<div class="flex justify-between border p-2 rounded">

<span>${c.code}</span>

<button onclick="removeCoupon('${c.code}')">X</button>

</div>

`).join("")

return`

<div class="p-6 space-y-6">

<h1 class="text-xl font-bold">Painel Admin</h1>

<button onclick="logout()" class="bg-red-500 text-white px-3 py-1 rounded">Sair</button>

<h2 class="font-bold">Usuários</h2>

<div class="space-y-2">${users}</div>

<h2 class="font-bold">Planos</h2>

<div class="space-y-3">${plans}</div>

<button onclick="savePlans()" class="bg-green-500 text-white px-3 py-1 rounded">Salvar</button>

<h2 class="font-bold">Cupons</h2>

<div class="flex gap-2">

<input id="new-coupon-code" placeholder="Código" class="border p-1">

<input id="new-coupon-discount" type="number" placeholder="Valor" class="border p-1">

<button onclick="addCoupon()" class="bg-black text-white px-2">+</button>

</div>

<div class="space-y-2">${coupons}</div>

</div>

`

}









/* ============================
RENDER CLIENT
============================ */

function renderClient(){

const plans=state.plans.map(p=>`

<div onclick="selectPlan('${p.id}')" class="border p-4 rounded cursor-pointer">

<b>${p.title}</b>

<div>R$ ${p.price.toFixed(2)}</div>

</div>

`).join("")

return`

<div class="p-6 space-y-6">

<button onclick="logout()" class="bg-red-500 text-white px-3 py-1 rounded">Sair</button>

<h1 class="text-xl font-bold">Escolha um plano</h1>

<div class="grid grid-cols-2 gap-3">

${plans}

</div>

<button onclick="handleWhatsAppRedirect()" class="bg-green-500 text-white p-3 rounded w-full">

Comprar via WhatsApp

</button>

</div>

`

}









/* ============================
RENDER GERAL
============================ */

function render(){

const root=document.getElementById("app-root")

if(state.currentView==="login")
root.innerHTML=renderLogin()

if(state.currentView==="admin")
root.innerHTML=renderAdmin()

if(state.currentView==="client")
root.innerHTML=renderClient()

lucide.createIcons()

}









/* ============================
INICIAR APP
============================ */

window.onload=()=>{

loadDatabase()

render()

}

</script>

</body>
</html>
