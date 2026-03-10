<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>UNITV VIP</title>
    
    <!-- Tailwind CSS para Estilização -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Ícones Lucide -->
    <script src="https://unpkg.com/lucide@latest"></script>

    <style>
        /* Ocultar barra de rolagem horizontal mantendo a funcionalidade */
        .hide-scrollbar::-webkit-scrollbar { display: none; }
        .hide-scrollbar { -ms-overflow-style: none; scrollbar-width: none; }
        
        @keyframes fadeIn {
            from { opacity: 0; transform: translate(-50%, -10px); }
            to { opacity: 1; transform: translate(-50%, 0); }
        }
        .animate-fade-in { animation: fadeIn 0.3s ease-out forwards; }
    </style>
</head>
<body class="bg-gray-50 font-sans text-gray-800">

    <!-- Contentor para Notificações (Toasts) -->
    <div id="toast-container" class="fixed top-4 left-1/2 -translate-x-1/2 z-[100] w-max"></div>

    <!-- Contentor Principal da Aplicação -->
    <div id="app-root"></div>

    <script>
        // --- ESTADO GLOBAL DA APLICAÇÃO ---
        const state = {
            currentView: 'login', // 'login', 'admin', 'client'
            currentUser: null,
            isLoginMode: true,
            users: [],
            plans: [
                { id: 'annual', title: 'Anual: 365 dias', price: 249.90, oldPrice: 299.00, monthlyAvg: 20.54, stock: 10 },
                { id: 'monthly', title: 'Mensal', price: 39.90, oldPrice: 49.00, monthlyAvg: 39.90, stock: 2 },
                { id: 'quarterly', title: 'Trimestral', price: 99.90, oldPrice: 139.00, monthlyAvg: 33.30, stock: 5 }
            ],
            coupons: [
                { code: 'VIP10', discount: 10.00 },
                { code: 'OFERTA5', discount: 5.00 }
            ],
            selectedPlanId: 'monthly',
            appliedCoupon: null,
            couponInput: ''
        };

        // --- SISTEMA DE NOTIFICAÇÕES (TOAST) ---
        function showToast(message, type = 'success') {
            const container = document.getElementById('toast-container');
            const bgColor = type === 'error' ? 'bg-red-500' : 'bg-gray-800';
            const iconName = type === 'error' ? 'alert-circle' : 'check-circle-2';
            
            const toastEl = document.createElement('div');
            toastEl.className = `px-6 py-3 rounded-full shadow-lg text-white text-sm font-medium flex items-center gap-2 mb-2 animate-fade-in ${bgColor}`;
            toastEl.innerHTML = `<i data-lucide="${iconName}" class="w-4 h-4"></i> <span>${message}</span>`;
            
            container.appendChild(toastEl);
            lucide.createIcons(); // Renderiza o ícone do toast

            setTimeout(() => {
                toastEl.remove();
            }, 3000);
        }

        // --- FUNÇÕES GLOBAIS DE ACÇÃO ---
        function toggleLoginMode() {
            state.isLoginMode = !state.isLoginMode;
            render();
        }

        function handleAuth(event) {
            event.preventDefault();
            const idInput = document.getElementById('auth-id').value.trim();
            const passInput = document.getElementById('auth-pass').value.trim();

            if (!idInput || !passInput) {
                showToast('Preencha ID e Senha', 'error');
                return;
            }

            if (state.isLoginMode) {
                // Login
                if (idInput === 'admin' && passInput === 'admin') {
                    state.currentUser = { id: 'Admin', role: 'admin' };
                    state.currentView = 'admin';
                    showToast('Bem-vindo, Administrador!');
                    render();
                } else {
                    const foundUser = state.users.find(u => u.id === idInput && u.password === passInput);
                    if (foundUser) {
                        state.currentUser = { id: idInput, role: 'client' };
                        state.currentView = 'client';
                        state.selectedPlanId = 'monthly'; // Reset plan selection
                        state.appliedCoupon = null; // Reset coupon
                        showToast(`Bem-vindo de volta, ${idInput}!`);
                        render();
                    } else {
                        showToast('ID ou senha incorretos!', 'error');
                    }
                }
            } else {
                // Registo
                if (idInput.toLowerCase() === 'admin') {
                    showToast('O ID "admin" é reservado.', 'error');
                    return;
                }
                const idExists = state.users.some(u => u.id === idInput);
                if (idExists) {
                    showToast('Este ID já está em uso!', 'error');
                    return;
                }
                state.users.push({ id: idInput, password: passInput });
                showToast('Conta criada com sucesso! Faça login.', 'success');
                state.isLoginMode = true;
                render();
            }
        }

        function logout() {
            state.currentUser = null;
            state.currentView = 'login';
            state.isLoginMode = true;
            render();
        }

        // Funções Admin
        function savePlans() {
            state.plans.forEach(plan => {
                const price = parseFloat(document.getElementById(`price-${plan.id}`).value);
                const oldPrice = parseFloat(document.getElementById(`oldprice-${plan.id}`).value);
                const stock = parseInt(document.getElementById(`stock-${plan.id}`).value);
                
                plan.price = isNaN(price) ? plan.price : price;
                plan.oldPrice = isNaN(oldPrice) ? plan.oldPrice : oldPrice;
                plan.stock = isNaN(stock) ? plan.stock : stock;
            });
            showToast('Preços e stock atualizados com sucesso!');
            render();
        }

        function addCoupon() {
            const code = document.getElementById('new-coupon-code').value.trim().toUpperCase();
            const discount = parseFloat(document.getElementById('new-coupon-discount').value);

            if (!code || isNaN(discount)) {
                showToast('Preencha código e desconto válido', 'error');
                return;
            }
            state.coupons.push({ code, discount });
            showToast('Cupom adicionado!');
            render();
        }

        function removeCoupon(code) {
            state.coupons = state.coupons.filter(c => c.code !== code);
            showToast('Cupom removido!');
            render();
        }

        // Funções Cliente
        function selectPlan(id) {
            const plan = state.plans.find(p => p.id === id);
            if (plan && plan.stock > 0) {
                state.selectedPlanId = id;
                render();
            }
        }

        function applyCoupon() {
            const input = document.getElementById('client-coupon-input').value.trim().toUpperCase();
            if (!input) return;

            const found = state.coupons.find(c => c.code === input);
            if (found) {
                state.appliedCoupon = found;
                showToast('Cupom aplicado com sucesso!');
            } else {
                showToast('Cupom inválido ou expirado.', 'error');
                state.appliedCoupon = null;
            }
            render();
        }

        function removeAppliedCoupon() {
            state.appliedCoupon = null;
            showToast('Cupom removido.');
            render();
        }

        function handleWhatsAppRedirect() {
            const phoneNumber = "5598984533013"; // O seu número
            const plan = state.plans.find(p => p.id === state.selectedPlanId);
            
            let finalPrice = plan.price;
            if (state.appliedCoupon) {
                finalPrice = Math.max(0, finalPrice - state.appliedCoupon.discount);
            }

            let message = `Olá! Meu ID de acesso é *${state.currentUser.id}*.\n\n`;
            message += `Gostaria de assinar o plano: *${plan.title}*\n`;
            if (state.appliedCoupon) {
                message += `🎁 Cupom aplicado: *${state.appliedCoupon.code}* (-R$ ${state.appliedCoupon.discount.toFixed(2)})\n`;
            }
            message += `💰 Valor total a pagar: *R$ ${finalPrice.toFixed(2)}*`;

            const encodedMessage = encodeURIComponent(message);
            window.open(`https://wa.me/${phoneNumber}?text=${encodedMessage}`, '_blank');
        }

        // --- RENDERIZAÇÃO DE INTERFACES ---
        
        function renderLogin() {
            return `
            <div class="min-h-screen bg-gray-50 flex flex-col justify-center p-6 md:max-w-md md:mx-auto md:border md:shadow-lg relative">
                <div class="bg-white rounded-2xl shadow-xl p-8 space-y-6">
                    <div class="text-center">
                        <div class="bg-[#effef5] w-16 h-16 rounded-full flex items-center justify-center mx-auto mb-4">
                            <i data-lucide="${state.isLoginMode ? 'log-in' : 'user-plus'}" class="w-8 h-8 text-[#5cb85c]"></i>
                        </div>
                        <h1 class="text-2xl font-bold text-gray-800">${state.isLoginMode ? 'Acessar Conta' : 'Criar Nova Conta'}</h1>
                        <p class="text-gray-500 text-sm mt-2">${state.isLoginMode ? 'Insira seus dados para continuar' : 'Crie seu ID exclusivo e senha'}</p>
                    </div>
                    
                    <form onsubmit="handleAuth(event)" class="space-y-4">
                        <div>
                            <label class="text-sm font-medium text-gray-700 block mb-1">ID do Usuário</label>
                            <div class="relative">
                                <i data-lucide="user" class="w-5 h-5 text-gray-400 absolute left-3 top-1/2 -translate-y-1/2"></i>
                                <input id="auth-id" type="text" class="w-full pl-10 pr-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#5cb85c] focus:border-[#5cb85c] outline-none transition-all" placeholder="Seu ID (ex: 123456)">
                            </div>
                        </div>
                        
                        <div>
                            <label class="text-sm font-medium text-gray-700 block mb-1">Senha</label>
                            <div class="relative">
                                <i data-lucide="lock" class="w-5 h-5 text-gray-400 absolute left-3 top-1/2 -translate-y-1/2"></i>
                                <input id="auth-pass" type="password" class="w-full pl-10 pr-4 py-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-[#5cb85c] focus:border-[#5cb85c] outline-none transition-all" placeholder="${state.isLoginMode ? 'Sua senha' : 'Crie uma senha forte'}">
                            </div>
                        </div>

                        <button type="submit" class="w-full bg-[#6cc57b] hover:bg-[#5bb56a] text-white font-bold py-3 rounded-lg transition-colors mt-2">
                            ${state.isLoginMode ? 'Entrar' : 'Criar Conta'}
                        </button>
                    </form>

                    <div class="text-center pt-4 border-t">
                        <button type="button" onclick="toggleLoginMode()" class="text-sm text-gray-600 hover:text-[#5cb85c] font-medium">
                            ${state.isLoginMode ? 'Não tem uma conta? Crie uma aqui' : 'Já tem uma conta? Fazer Login'}
                        </button>
                    </div>
                </div>
            </div>`;
        }

        function renderAdmin() {
            let usersHTML = state.users.length === 0 
                ? '<p class="text-sm text-gray-500 text-center py-2">Nenhum cliente cadastrado ainda.</p>'
                : state.users.map(u => `
                    <div class="flex items-center justify-between p-2 border rounded bg-gray-50">
                        <span class="font-bold text-gray-700">ID: ${u.id}</span>
                        <span class="text-xs text-gray-400">Senha: ${u.password}</span>
                    </div>`).join('');

            let plansHTML = state.plans.map(plan => `
                <div class="p-3 border rounded-lg bg-gray-50">
                    <span class="font-bold text-gray-700 block mb-2">${plan.title}</span>
                    <div class="grid grid-cols-3 gap-2">
                        <div>
                            <label class="text-xs text-gray-500">Preço (R$)</label>
                            <input id="price-${plan.id}" type="number" step="0.01" value="${plan.price}" class="w-full p-2 border rounded focus:border-[#6cc57b] outline-none text-sm">
                        </div>
                        <div>
                            <label class="text-xs text-gray-500">Antigo (R$)</label>
                            <input id="oldprice-${plan.id}" type="number" step="0.01" value="${plan.oldPrice}" class="w-full p-2 border rounded focus:border-[#6cc57b] outline-none text-sm">
                        </div>
                        <div>
                            <label class="text-xs text-gray-500">Estoque</label>
                            <input id="stock-${plan.id}" type="number" value="${plan.stock}" class="w-full p-2 border rounded focus:border-[#6cc57b] outline-none text-sm">
                        </div>
                    </div>
                </div>`).join('');

            let couponsHTML = state.coupons.length === 0 
                ? '<p class="text-sm text-gray-500 text-center py-2">Nenhum cupom ativo.</p>'
                : state.coupons.map(c => `
                    <div class="flex items-center justify-between p-2 border border-dashed border-gray-300 rounded-lg bg-gray-50">
                        <div>
                            <span class="font-bold text-gray-800 block">${c.code}</span>
                            <span class="text-xs text-[#de5d5d] font-medium">- R$ ${c.discount.toFixed(2)}</span>
                        </div>
                        <button onclick="removeCoupon('${c.code}')" class="text-red-400 hover:text-red-600 p-1">
                            <i data-lucide="trash-2" class="w-4 h-4"></i>
                        </button>
                    </div>`).join('');

            return `
            <div class="min-h-screen bg-gray-100 pb-24 md:max-w-md md:mx-auto md:border md:shadow-lg relative overflow-y-auto">
                <header class="bg-[#1c1c24] text-white flex items-center justify-between p-4 sticky top-0 z-10">
                    <div class="flex items-center gap-2">
                        <i data-lucide="settings" class="w-6 h-6 text-[#5cb85c]"></i>
                        <h1 class="text-lg font-medium">Painel Admin</h1>
                    </div>
                    <button onclick="logout()" class="p-2 bg-gray-800 rounded-full hover:bg-gray-700">
                        <i data-lucide="log-out" class="w-5 h-5 text-gray-300"></i>
                    </button>
                </header>

                <main class="p-4 space-y-6">
                    <section class="bg-white p-4 rounded-xl shadow-sm border border-gray-200">
                        <h2 class="font-bold text-gray-800 text-lg flex items-center gap-2 mb-3">
                            <i data-lucide="user" class="w-5 h-5 text-blue-500"></i> Clientes Cadastrados
                        </h2>
                        <div class="space-y-2">${usersHTML}</div>
                    </section>

                    <section class="bg-white p-4 rounded-xl shadow-sm border border-gray-200">
                        <div class="flex items-center justify-between mb-4">
                            <h2 class="font-bold text-gray-800 text-lg flex items-center gap-2">
                                <i data-lucide="tag" class="w-5 h-5 text-[#6cc57b]"></i> Editar Preços
                            </h2>
                            <button onclick="savePlans()" class="flex items-center gap-1 bg-[#6cc57b] text-white px-3 py-1.5 rounded text-sm font-medium hover:bg-[#5bb56a]">
                                <i data-lucide="save" class="w-4 h-4"></i> Salvar
                            </button>
                        </div>
                        <div class="space-y-4">${plansHTML}</div>
                    </section>

                    <section class="bg-white p-4 rounded-xl shadow-sm border border-gray-200">
                        <h2 class="font-bold text-gray-800 text-lg flex items-center gap-2 mb-4">
                            <i data-lucide="ticket" class="w-5 h-5 text-[#de5d5d]"></i> Gerenciar Cupons
                        </h2>
                        <div class="flex gap-2 mb-4">
                            <input id="new-coupon-code" type="text" placeholder="CÓDIGO" class="flex-1 p-2 border rounded text-sm uppercase outline-none focus:border-[#6cc57b]">
                            <input id="new-coupon-discount" type="number" placeholder="Valor (R$)" class="w-24 p-2 border rounded text-sm outline-none focus:border-[#6cc57b]">
                            <button onclick="addCoupon()" class="bg-gray-800 text-white p-2 rounded hover:bg-gray-700">
                                <i data-lucide="plus" class="w-5 h-5"></i>
                            </button>
                        </div>
                        <div class="space-y-2">${couponsHTML}</div>
                    </section>
                </main>
            </div>`;
        }

        function renderClient() {
            const features = [
                "2 Dispositivos Simultâneos: TV, Celular ou Web",
                "Canais ao vivo: +500",
                "UHD (1080P): +50",
                "VOD: +1 Milhão",
                "+18: Canais+Vod"
            ];

            const featuresHTML = features.map(feature => `
                <div class="flex items-start gap-3">
                    <i data-lucide="check-circle-2" class="w-5 h-5 text-[#5cb85c] flex-shrink-0 mt-0.5 fill-current text-white"></i>
                    <span class="text-[15px] font-bold text-gray-500">${feature}</span>
                </div>
            `).join('');

            const plansHTML = state.plans.map(plan => {
                const isSelected = state.selectedPlanId === plan.id;
                const isOutOfStock = plan.stock === 0;
                
                let classes = `flex-shrink-0 w-36 rounded-xl border-2 flex flex-col overflow-hidden transition-all snap-center `;
                if (isOutOfStock) {
                    classes += `opacity-50 cursor-not-allowed bg-gray-100 border-gray-200`;
                } else if (isSelected) {
                    classes += `border-[#a3e4bd] bg-[#effef5] cursor-pointer`;
                } else {
                    classes += `border-transparent bg-[#f7f7f7] cursor-pointer`;
                }

                let badgeHTML = plan.stock > 0 
                    ? `<span class="text-[10px] font-bold text-[#5cb85c] bg-[#5cb85c]/10 px-2 py-0.5 rounded-full mb-2">${plan.stock} disponíveis</span>`
                    : `<span class="text-[10px] font-bold text-red-500 bg-red-500/10 px-2 py-0.5 rounded-full mb-2">Esgotado</span>`;

                return `
                <div onclick="selectPlan('${plan.id}')" class="${classes}">
                    <div class="p-4 flex flex-col items-center flex-1 text-center">
                        <h3 class="text-[13px] font-bold text-black mb-1">${plan.title}</h3>
                        ${badgeHTML}
                        <div class="flex items-baseline justify-center gap-1 mb-1">
                            <span class="text-[#e29947] font-bold text-sm">R$</span>
                            <span class="text-[#e29947] font-bold text-2xl tracking-tighter">${plan.price.toFixed(2)}</span>
                        </div>
                        <span class="text-gray-400 text-xs line-through mb-1">R$ ${plan.oldPrice.toFixed(2)}</span>
                        <span class="text-gray-500 text-[11px]">R$ ${plan.monthlyAvg.toFixed(2)}/mês</span>
                    </div>
                </div>`;
            }).join('');

            const selectedPlan = state.plans.find(p => p.id === state.selectedPlanId);
            let finalPrice = selectedPlan ? selectedPlan.price : 0;
            if (state.appliedCoupon) {
                finalPrice = Math.max(0, finalPrice - state.appliedCoupon.discount);
            }

            const couponSectionHTML = !state.appliedCoupon ? `
                <div class="flex gap-2">
                    <input id="client-coupon-input" type="text" placeholder="Digite o código" class="flex-1 border border-gray-300 rounded-lg px-3 py-2 text-sm uppercase outline-none focus:border-[#5cb85c]">
                    <button onclick="applyCoupon()" class="bg-gray-800 text-white px-4 py-2 rounded-lg text-sm font-medium hover:bg-gray-700 transition-colors">Aplicar</button>
                </div>
            ` : `
                <div class="bg-[#de5d5d]/10 border border-[#de5d5d]/30 rounded-lg p-3 flex justify-between items-center">
                    <div>
                        <span class="block text-xs text-gray-500">Cupom aplicado:</span>
                        <span class="font-bold text-[#de5d5d]">${state.appliedCoupon.code} (-R$ ${state.appliedCoupon.discount.toFixed(2)})</span>
                    </div>
                    <button onclick="removeAppliedCoupon()" class="text-gray-400 hover:text-gray-600">
                        <i data-lucide="x" class="w-5 h-5"></i>
                    </button>
                </div>
            `;

            return `
            <div class="min-h-screen bg-white font-sans text-gray-800 pb-32 md:max-w-md md:mx-auto md:border md:border-gray-200 md:shadow-lg relative">
                <header class="bg-[#1c1c24] text-white flex items-center p-4 sticky top-0 z-10">
                    <i data-lucide="chevron-left" class="w-6 h-6 mr-4 cursor-pointer text-gray-300"></i>
                    <h1 class="flex-1 text-center text-lg font-medium">Seja um membro VIP</h1>
                    <button onclick="logout()" class="ml-4 flex items-center gap-1" title="Sair">
                        <span class="text-xs text-gray-400 hidden sm:block">${state.currentUser.id}</span>
                        <i data-lucide="log-out" class="w-5 h-5 text-gray-300 hover:text-white"></i>
                    </button>
                </header>

                <main class="p-4 space-y-6">
                    <div class="bg-[#f0f0f0] rounded-xl p-3 flex items-center gap-3">
                        <div class="bg-[#facc15] rounded-full p-0.5">
                            <i data-lucide="alert-circle" class="w-4 h-4 text-white fill-current"></i>
                        </div>
                        <span class="text-sm font-medium text-gray-700">Pode assistir na TV + Mobile</span>
                    </div>

                    <div class="border-b border-gray-200">
                        <div class="inline-block border-b-2 border-[#5cb85c] pb-2 px-1">
                            <span class="text-[#5cb85c] font-bold text-lg tracking-wide">UNITV VIP</span>
                        </div>
                    </div>

                    <div class="space-y-4 pt-2">
                        ${featuresHTML}
                    </div>

                    <div class="flex gap-3 overflow-x-auto pb-4 snap-x hide-scrollbar pt-2" style="scrollbar-width: none; -ms-overflow-style: none;">
                        ${plansHTML}
                    </div>

                    <div class="bg-gray-50 p-4 rounded-xl border border-gray-200 mt-4">
                        <h3 class="text-sm font-bold text-gray-700 mb-3 flex items-center gap-2">
                            <i data-lucide="ticket" class="w-4 h-4 text-[#de5d5d]"></i> Possui um cupom?
                        </h3>
                        ${couponSectionHTML}
                    </div>
                </main>

                <footer class="fixed bottom-0 w-full bg-white border-t border-gray-200 p-3 flex justify-between items-center md:max-w-md shadow-[0_-4px_6px_-1px_rgba(0,0,0,0.1)] z-50">
                    <div class="flex items-end gap-3">
                        <div class="flex flex-col">
                            ${state.appliedCoupon ? `<span class="text-xs text-gray-400 line-through mb-0.5">R$ ${selectedPlan.price.toFixed(2)}</span>` : ''}
                            <div class="flex items-baseline text-[#de5d5d]">
                                <span class="font-bold text-sm">R$</span>
                                <span class="font-bold text-3xl tracking-tighter">${finalPrice.toFixed(2)}</span>
                            </div>
                        </div>
                        <div class="flex flex-col pb-1">
                            <span class="text-sm font-bold text-black">UNITV VIP</span>
                            <span class="text-sm font-bold text-black leading-tight">${selectedPlan ? selectedPlan.title.split(':')[0] : ''}</span>
                        </div>
                    </div>
                    
                    <button 
                        ${selectedPlan && selectedPlan.stock === 0 ? 'disabled' : ''}
                        onclick="handleWhatsAppRedirect()"
                        class="font-medium py-3 px-8 rounded-lg text-lg transition-colors shadow-sm ${selectedPlan && selectedPlan.stock === 0 ? 'bg-gray-400 cursor-not-allowed text-gray-200' : 'bg-[#6cc57b] hover:bg-[#5bb56a] text-white'}">
                        Comprar
                    </button>
                </footer>
            </div>`;
        }

        // --- MOTOR DE RENDERIZAÇÃO ---
        function render() {
            const root = document.getElementById('app-root');
            if (state.currentView === 'login') {
                root.innerHTML = renderLogin();
            } else if (state.currentView === 'admin') {
                root.innerHTML = renderAdmin();
            } else if (state.currentView === 'client') {
                root.innerHTML = renderClient();
            }
            
            // Renderiza os ícones SVG do Lucide após atualizar o HTML
            lucide.createIcons();
        }

        // Inicia a aplicação
        window.onload = render;

    </script>
</body>
</html>

