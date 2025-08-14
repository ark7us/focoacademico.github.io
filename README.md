        switch (anotacao.tipo) {
            case 'texto': return <FileText className="w-5 h-5 text-blue-500" />;
            case 'foto': return <Camera className="w-5 h-5 text-purple-500" />;
            default: return <FileText className="w-5 h-5 text-gray-500" />;
        }
    };
    return (
        <div className="bg-white p-3 rounded-lg shadow-sm flex items-start space-x-3 relative group">
            <div className="flex-shrink-0">{getIcone()}</div>
            <div className="flex-1">
                <p className="text-sm text-gray-800 whitespace-pre-wrap">{anotacao.conteudo}</p>
                <p className="text-xs text-gray-400 mt-1">{new Date(anotacao.data).toLocaleString('pt-BR')}</p>
                {anotacao.tipo === 'foto' && anotacao.arquivoUrl && <img src={anotacao.arquivoUrl} alt="Preview" className="mt-2 rounded-lg max-w-full h-auto" />}
            </div>
            <button onClick={onDelete} className="absolute top-2 right-2 text-gray-400 hover:text-red-600 opacity-0 group-hover:opacity-100 transition-opacity"><Trash2 size={16} /></button>
        </div>
    );
};

const Cronometro = ({ onComplete }) => {
    const TEMPO_FOCO = 15 * 60; // 15 minutos
    const [segundos, setSegundos] = useState(TEMPO_FOCO);
    const [ativo, setAtivo] = useState(false);

    useEffect(() => {
        let intervalo = null;
        if (ativo && segundos > 0) {
            intervalo = setInterval(() => {
                setSegundos(s => s - 1);
            }, 1000);
        } else if (segundos === 0) {
            clearInterval(intervalo);
            onComplete(TEMPO_FOCO / 60);
            setAtivo(false);
            alert("Sessão de foco concluída! Bom trabalho.");
        }
        return () => clearInterval(intervalo);
    }, [ativo, segundos, onComplete, TEMPO_FOCO]);

    const toggle = () => setAtivo(!ativo);
    const reset = () => {
        setAtivo(false);
        setSegundos(TEMPO_FOCO);
    };

    return (
        <div className="bg-blue-50 p-4 rounded-xl shadow-inner text-center">
            <h3 className="font-bold text-blue-800 mb-2">Cronômetro de Foco</h3>
            <div className="text-5xl font-mono text-blue-900 mb-4">
                {Math.floor(segundos / 60).toString().padStart(2, '0')}:
                {(segundos % 60).toString().padStart(2, '0')}
            </div>
            <div className="flex justify-center space-x-4">
                <button onClick={toggle} className="bg-blue-500 text-white p-3 rounded-full shadow-lg">
                    {ativo ? <Pause size={20} /> : <Play size={20} />}
                </button>
                <button onClick={reset} className="bg-gray-300 text-gray-800 p-3 rounded-full shadow-lg">
                    <RefreshCw size={20} />
                </button>
            </div>
        </div>
    );
};

// --- COMPONENTE PRINCIPAL ---

export default function App() {
    const [tela, setTela] = useState('home');
    const [subTela, setSubTela] = useState(null);
    const [itemAtivo, setItemAtivo] = useState(null);

    const carregarDoLocalStorage = useCallback((chave, valorPadrao) => {
        try {
            const item = window.localStorage.getItem(chave);
            return item ? JSON.parse(item) : valorPadrao;
        } catch (error) {
            console.error(`Erro ao carregar ${chave}`, error);
            return valorPadrao;
        }
    }, []);

    const [disciplinas, setDisciplinas] = useState(() => carregarDoLocalStorage(LOCAL_STORAGE_KEYS.DISCIPLINAS, []));
    const [eventos, setEventos] = useState(() => carregarDoLocalStorage(LOCAL_STORAGE_KEYS.EVENTOS, []));
    const [tempoEstudoTotal, setTempoEstudoTotal] = useState(() => carregarDoLocalStorage(LOCAL_STORAGE_KEYS.TEMPO_ESTUDO, 0));

    useEffect(() => { window.localStorage.setItem(LOCAL_STORAGE_KEYS.DISCIPLINAS, JSON.stringify(disciplinas)); }, [disciplinas]);
    useEffect(() => { window.localStorage.setItem(LOCAL_STORAGE_KEYS.EVENTOS, JSON.stringify(eventos)); }, [eventos]);
    useEffect(() => { window.localStorage.setItem(LOCAL_STORAGE_KEYS.TEMPO_ESTUDO, JSON.stringify(tempoEstudoTotal)); }, [tempoEstudoTotal]);

    // --- FUNÇÕES CRUD ---
    const salvarDisciplina = (disciplina) => {
        if (disciplina.id) {
            setDisciplinas(disciplinas.map(d => d.id === disciplina.id ? disciplina : d));
        } else {
            const novaDisciplina = { ...disciplina, id: Date.now(), anotacoes: [] };
            setDisciplinas([...disciplinas, novaDisciplina]);
        }
        setSubTela(null);
    };

    const salvarEvento = (evento) => {
        if (evento.id) {
            setEventos(eventos.map(e => e.id === evento.id ? evento : e));
        } else {
            const novoEvento = { ...evento, id: Date.now(), concluido: false };
            setEventos([...eventos, novoEvento]);
        }
        setSubTela(null);
    };
    
    const adicionarAnotacao = (disciplinaId, tipo, conteudo, arquivo) => {
        let arquivoUrl = null;
        if (arquivo && tipo === 'foto') {
            arquivoUrl = URL.createObjectURL(arquivo);
        }
        const novaAnotacao = { id: Date.now(), tipo, conteudo: arquivo ? arquivo.name : conteudo, arquivoUrl, data: new Date().toISOString() };
        setDisciplinas(disciplinas.map(d => d.id === disciplinaId ? { ...d, anotacoes: [novaAnotacao, ...d.anotacoes] } : d));
    };

    const deletarAnotacao = (disciplinaId, anotacaoId) => {
        setDisciplinas(disciplinas.map(d => {
            if (d.id === disciplinaId) {
                const anotacao = d.anotacoes.find(a => a.id === anotacaoId);
                if (anotacao?.arquivoUrl) URL.revokeObjectURL(anotacao.arquivoUrl);
                return { ...d, anotacoes: d.anotacoes.filter(a => a.id !== anotacaoId) };
            }
            return d;
        }));
    };
    
    const deletarEvento = (eventoId) => setEventos(eventos.filter(e => e.id !== eventoId));
    
    const deletarDisciplina = (disciplinaId) => {
        if(window.confirm("Tem certeza que deseja apagar esta disciplina e todos os seus eventos e anotações?")){
            setDisciplinas(disciplinas.filter(d => d.id !== disciplinaId));
            setEventos(eventos.filter(e => e.disciplinaId !== disciplinaId));
            setSubTela(null);
        }
    };

    const marcarEventoComoConcluido = (eventoId) => {
        setEventos(eventos.map(e => e.id === eventoId ? { ...e, concluido: !e.concluido } : e));
    };
    
    const adicionarTempoEstudo = useCallback((minutos) => {
        setTempoEstudoTotal(prev => prev + minutos);
    }, []);

    // --- NAVEGAÇÃO ---
    const irPara = (telaDestino) => setTela(telaDestino);
    const abrirSubTela = (nome, item = null) => {
        setItemAtivo(item);
        setSubTela(nome);
    };
    const fecharSubTela = () => {
        setItemAtivo(null);
        setSubTela(null);
    };

    // --- RENDERIZAÇÃO DAS TELAS ---

    const TelaHome = () => {
        const hojeString = new Date().toLocaleDateString('en-CA');
        const eventosDeHoje = useMemo(() => eventos.filter(e => e.data === hojeString).sort((a, b) => a.titulo.localeCompare(b.titulo)), [eventos, hojeString]);

        const aulasDeHoje = useMemo(() => {
            const dias = ['dom', 'seg', 'ter', 'qua', 'qui', 'sex', 'sab'];
            const hoje = dias[new Date().getDay()];
            return disciplinas
                .filter(d => d.horarios?.dias.includes(hoje))
                .sort((a,b) => (a.horarios?.inicio || '').localeCompare(b.horarios?.inicio || ''));
        }, [disciplinas]);

        return (
            <div className="p-4 space-y-6">
                <h1 className="text-2xl font-bold text-gray-800">Foco Acadêmico</h1>
                
                <div className="bg-white p-4 rounded-xl shadow-md">
                    <h2 className="font-bold text-gray-700 mb-3 flex items-center"><Clock size={20} className="mr-2" /> Aulas de Hoje</h2>
                    {aulasDeHoje.length > 0 ? (
                        <ul className="space-y-2">
                            {aulasDeHoje.map(d => (
                                <li key={d.id} className="flex items-center justify-between p-2 rounded-lg bg-blue-50">
                                    <p className="font-semibold text-sm text-blue-800">{d.nome}</p>
                                    <p className="text-xs text-blue-700 font-mono">{d.horarios.inicio} - {d.horarios.fim}</p>
                                </li>
                            ))}
                        </ul>
                    ) : <p className="text-sm text-gray-500 text-center py-2">Nenhuma aula hoje.</p>}
                </div>

                <div className="bg-white p-4 rounded-xl shadow-md">
                    <h2 className="font-bold text-gray-700 mb-3 flex items-center"><Calendar size={20} className="mr-2" /> Tarefas e Provas de Hoje</h2>
                    {eventosDeHoje.length > 0 ? (
                        <ul className="space-y-2">
                            {eventosDeHoje.map(e => {
                                const disciplina = disciplinas.find(d => d.id === e.disciplinaId);
                                return (
                                    <li key={e.id} onClick={() => marcarEventoComoConcluido(e.id)} className={`flex items-center justify-between p-2 rounded-lg cursor-pointer ${e.concluido ? 'bg-green-50 text-gray-500 line-through' : 'bg-gray-50'}`}>
                                        <div className="flex items-center"><div className="ml-2"><p className="font-semibold text-sm">{e.titulo}</p><p className="text-xs text-gray-500">{disciplina?.nome || 'Geral'}</p></div></div>
                                    </li>
                                );
                            })}
                        </ul>
                    ) : <p className="text-sm text-gray-500 text-center py-2">Nenhuma atividade para hoje.</p>}
                </div>

                <div>
                    <div className="flex justify-between items-center mb-3">
                         <h2 className="font-bold text-gray-700 flex items-center"><Book size={20} className="mr-2" /> Minhas Disciplinas</h2>
                         <button onClick={() => abrirSubTela('editarDisciplina')} className="bg-blue-500 text-white rounded-full p-2 shadow-lg"><Plus size={16}/></button>
                    </div>
                    <div className="space-y-3">
                        {disciplinas.map(d => (
                            <div key={d.id} onClick={() => abrirSubTela('disciplina', d)} className="bg-white p-4 rounded-xl shadow-sm hover:shadow-lg transition-shadow cursor-pointer flex items-center justify-between">
                                <span className="font-semibold text-gray-800">{d.nome}</span>
                                <ChevronRight size={20} className="text-gray-400" />
                            </div>
                        ))}
                    </div>
                </div>
            </div>
        );
    };

    const TelaDisciplina = () => {
        const disciplina = itemAtivo;
        const [formAberto, setFormAberto] = useState(null);
        const [conteudoAnotacao, setConteudoAnotacao] = useState('');
        const [tipoAnotacao, setTipoAnotacao] = useState('texto');
        const [arquivoAnotacao, setArquivoAnotacao] = useState(null);

        const handleAdicionarAnotacao = (e) => {
            e.preventDefault();
            if (conteudoAnotacao.trim() || arquivoAnotacao) {
                adicionarAnotacao(disciplina.id, tipoAnotacao, conteudoAnotacao, arquivoAnotacao);
                setConteudoAnotacao(''); setArquivoAnotacao(null); setFormAberto(null);
            }
        };
        
        const eventosDaDisciplina = useMemo(() => eventos.filter(e => e.disciplinaId === disciplina.id).sort((a, b) => new Date(a.data) - new Date(b.data)), [eventos, disciplina.id]);
        if (!disciplina) return null;
        
        const { nome, horarios } = disciplina;

        return (
            <div className="p-4 space-y-4">
                <button onClick={fecharSubTela} className="flex items-center text-gray-600 mb-2"><ChevronLeft size={20} /> Voltar</button>
                <div className="flex items-center justify-between">
                    <h1 className="text-2xl font-bold text-gray-800">{nome}</h1>
                    <div className="flex items-center gap-3">
                        <button onClick={() => abrirSubTela('editarDisciplina', disciplina)} className="text-gray-500 hover:text-blue-600"><Pencil size={18} /></button>
                        <button onClick={() => deletarDisciplina(disciplina.id)} className="text-gray-500 hover:text-red-600"><Trash2 size={18} /></button>
                    </div>
                </div>
                
                {horarios && horarios.dias.length > 0 && (
                    <div className="text-sm text-gray-700 bg-blue-50 p-3 rounded-lg">
                        <p className="font-bold text-blue-800">Horário da Aula:</p>
                        <p className="text-blue-700">
                            {horarios.dias.map(d => d.charAt(0).toUpperCase() + d.slice(1)).join(', ')}
                            {horarios.inicio && ` das ${horarios.inicio} às ${horarios.fim}`}
                        </p>
                    </div>
                )}

                <Cronometro onComplete={adicionarTempoEstudo} />

                <div>
                    <div className="flex justify-between items-center mb-2">
                        <h3 className="font-bold text-gray-700 flex items-center"><Calendar size={18} className="mr-2"/> Eventos</h3>
                        <button onClick={() => abrirSubTela('editarEvento', { disciplinaId: disciplina.id })} className="text-blue-500 text-sm font-semibold">Adicionar</button>
                    </div>
                     {eventosDaDisciplina.length > 0 ? (
                        <ul className="space-y-2">
                            {eventosDaDisciplina.map(e => (
                                <li key={e.id} className={`p-3 rounded-lg shadow-sm flex justify-between items-start group ${e.concluido ? 'bg-gray-100' : 'bg-white'}`}>
                                    <div>
                                        <p className={`font-semibold text-sm ${e.concluido ? 'line-through text-gray-500' : ''}`}>{e.titulo}</p>
                                        <p className="text-xs text-gray-500">{new Date(e.data + 'T00:00:00').toLocaleDateString('pt-BR')}</p>
                                    </div>
                                    <div className="flex items-center gap-2 opacity-0 group-hover:opacity-100 transition-opacity">
                                        <button onClick={() => abrirSubTela('editarEvento', e)} className="text-gray-400 hover:text-blue-600"><Pencil size={16} /></button>
                                        <button onClick={() => deletarEvento(e.id)} className="text-gray-400 hover:text-red-600"><Trash2 size={16} /></button>
                                    </div>
                                </li>
                            ))}
                        </ul>
                    ) : <p className="text-sm text-gray-500 text-center py-2 bg-gray-50 rounded-lg">Nenhum evento agendado.</p>}
                </div>

                <div>
                    <div className="flex justify-between items-center mb-2">
                        <h3 className="font-bold text-gray-700 flex items-center"><FileText size={18} className="mr-2"/> Anotações</h3>
                        <button onClick={() => setFormAberto(!formAberto)} className="text-blue-500 text-sm font-semibold">{formAberto ? 'Cancelar' : 'Adicionar'}</button>
                    </div>
                    {formAberto && (
                        <form onSubmit={handleAdicionarAnotacao} className="bg-white p-4 rounded-lg shadow-md mb-4 space-y-3">
                            <div className="flex justify-around">
                                <button type="button" onClick={() => setTipoAnotacao('texto')} className={tipoAnotacao === 'texto' ? 'text-blue-600 font-bold' : 'text-gray-500'}><FileText /></button>
                                <button type="button" onClick={() => setTipoAnotacao('foto')} className={tipoAnotacao === 'foto' ? 'text-blue-600 font-bold' : 'text-gray-500'}><Camera /></button>
                            </div>
                            {tipoAnotacao === 'texto' ? <textarea value={conteudoAnotacao} onChange={e => setConteudoAnotacao(e.target.value)} placeholder="Digite sua anotação..." className="w-full border rounded p-2 h-24" />
                            : <input type="file" onChange={e => setArquivoAnotacao(e.target.files[0])} className="w-full text-sm" accept="image/*" />}
                            <button type="submit" className="w-full text-sm bg-blue-500 text-white px-3 py-2 rounded">Salvar Anotação</button>
                        </form>
                    )}
                    {disciplina.anotacoes.length > 0 ? (
                        <div className="space-y-2">{disciplina.anotacoes.map(a => <AnotacaoItem key={a.id} anotacao={a} onDelete={() => deletarAnotacao(disciplina.id, a.id)} />)}</div>
                    ) : <p className="text-sm text-gray-500 text-center py-2 bg-gray-50 rounded-lg">Nenhuma anotação ainda.</p>}
                </div>
            </div>
        );
    };

    const FormularioDisciplina = () => {
        const [disciplina, setDisciplina] = useState(itemAtivo || { nome: '', horarios: { dias: [], inicio: '', fim: '' } });
        const isEditing = !!itemAtivo;

        const DIAS_SEMANA = [
            { id: 'seg', nome: 'S' }, { id: 'ter', nome: 'T' }, { id: 'qua', nome: 'Q' },
            { id: 'qui', nome: 'Q' }, { id: 'sex', nome: 'S' }, { id: 'sab', nome: 'S' }
        ];

        const handleDiaChange = (diaId) => {
            const novosDias = disciplina.horarios.dias.includes(diaId)
                ? disciplina.horarios.dias.filter(d => d !== diaId)
                : [...disciplina.horarios.dias, diaId];
            setDisciplina({ ...disciplina, horarios: { ...disciplina.horarios, dias: novosDias } });
        };

        const handleHorarioChange = (e) => {
            setDisciplina({ ...disciplina, horarios: { ...disciplina.horarios, [e.target.name]: e.target.value } });
        };
        
        const handleNomeChange = (e) => {
            setDisciplina({ ...disciplina, nome: e.target.value });
        };

        const handleSubmit = (e) => { e.preventDefault(); if (disciplina.nome.trim()) salvarDisciplina(disciplina); };

        return (
            <div className="p-4">
                <button onClick={fecharSubTela} className="flex items-center text-gray-600 mb-4"><ChevronLeft size={20} /> Voltar</button>
                <h1 className="text-2xl font-bold mb-4">{isEditing ? 'Editar Disciplina' : 'Nova Disciplina'}</h1>
                <form onSubmit={handleSubmit} className="space-y-6">
                    <div>
                        <label htmlFor="nome" className="block text-sm font-medium text-gray-700">Nome da 
