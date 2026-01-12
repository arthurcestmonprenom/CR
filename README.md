<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Gestionnaire CR Chantier</title>
    
    <!-- React & ReactDOM -->
    <script src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
    <script src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
    
    <!-- Babel pour JSX -->
    <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <!-- Lucide Icons -->
    <script src="https://unpkg.com/lucide@latest"></script>
    
    <!-- Styles personnalisés pour l'impression -->
    <style>
        @media print {
            @page { margin: 10mm; size: A4; }
            body { -webkit-print-color-adjust: exact; background: white; height: auto !important; overflow: visible !important; }
            #root { display: none; } /* Cache l'interface React normale */
            #print-area { display: block !important; position: absolute; top: 0; left: 0; width: 100%; }
            .print-hidden { display: none !important; }
            .avoid-break { break-inside: avoid; }
        }
        #print-area { display: none; } /* Cache la zone d'impression à l'écran */
        
        /* Scrollbar custom */
        ::-webkit-scrollbar { width: 8px; }
        ::-webkit-scrollbar-track { background: #f1f1f1; }
        ::-webkit-scrollbar-thumb { background: #888; border-radius: 4px; }
        ::-webkit-scrollbar-thumb:hover { background: #555; }
    </style>
</head>
<body class="bg-gray-100 text-gray-900 font-sans h-screen overflow-hidden">
    <div id="root" class="h-full"></div>
    <div id="print-area"></div>

    <script type="text/babel">
        const { useState, useEffect, useRef } = React;
        const { 
            LayoutDashboard, Users, Calendar, MessageSquare, Printer, Save, Plus, 
            Trash2, CheckCircle2, AlertCircle, FileText, Building, ArrowLeft, 
            Settings, Clock, Briefcase, Camera, Image: ImageIcon, CloudRain, 
            AlertTriangle, ChevronDown, CheckSquare, Edit3, EyeOff, Archive,
            Palette, MoveUp, MoveDown, Eye, LayoutTemplate, HelpCircle, X, ExternalLink,
            Type, UserCircle, Sparkles, Loader2, UserPlus, FileEdit, RefreshCw, PenSquare,
            UserX, UserCheck, Bell, BellOff
        } = lucide;

        // --- CONFIGURATION API & DB ---
        const apiKey = ""; 

        // --- GESTION BASE DE DONNÉES (IndexedDB) ---
        const DB_NAME = 'CRChantierDB';
        const DB_VERSION = 1;
        const STORE_NAME = 'projects';

        const dbHelper = {
            open: () => {
                return new Promise((resolve, reject) => {
                    const request = indexedDB.open(DB_NAME, DB_VERSION);
                    request.onupgradeneeded = (event) => {
                        const db = event.target.result;
                        if (!db.objectStoreNames.contains(STORE_NAME)) {
                            db.createObjectStore(STORE_NAME, { keyPath: 'id' });
                        }
                    };
                    request.onsuccess = (event) => resolve(event.target.result);
                    request.onerror = (event) => reject(event.target.error);
                });
            },
            getAll: async () => {
                const db = await dbHelper.open();
                return new Promise((resolve, reject) => {
                    const transaction = db.transaction(STORE_NAME, 'readonly');
                    const store = transaction.objectStore(STORE_NAME);
                    const request = store.getAll();
                    request.onsuccess = () => resolve(request.result);
                    request.onerror = () => reject(request.error);
                });
            },
            save: async (project) => {
                const db = await dbHelper.open();
                return new Promise((resolve, reject) => {
                    const transaction = db.transaction(STORE_NAME, 'readwrite');
                    const store = transaction.objectStore(STORE_NAME);
                    const request = store.put(project);
                    request.onsuccess = () => resolve(request.result);
                    request.onerror = () => reject(request.error);
                });
            },
            delete: async (id) => {
                const db = await dbHelper.open();
                return new Promise((resolve, reject) => {
                    const transaction = db.transaction(STORE_NAME, 'readwrite');
                    const store = transaction.objectStore(STORE_NAME);
                    const request = store.delete(id);
                    request.onsuccess = () => resolve();
                    request.onerror = () => reject(request.error);
                });
            }
        };

        // --- UTILITAIRE IMAGE (Compression) ---
        const resizeImage = (file, maxWidth = 1024, quality = 0.7) => {
            return new Promise((resolve, reject) => {
                const reader = new FileReader();
                reader.readAsDataURL(file);
                reader.onload = (event) => {
                    const img = new Image();
                    img.src = event.target.result;
                    img.onload = () => {
                        const elem = document.createElement('canvas');
                        let width = img.width;
                        let height = img.height;

                        if (width > maxWidth) {
                            height *= maxWidth / width;
                            width = maxWidth;
                        }

                        elem.width = width;
                        elem.height = height;
                        const ctx = elem.getContext('2d');
                        ctx.drawImage(img, 0, 0, width, height);
                        resolve(ctx.canvas.toDataURL(file.type, quality));
                    };
                    img.onerror = (err) => reject(err);
                };
                reader.onerror = (err) => reject(err);
            });
        };

        const callGemini = async (prompt) => {
            try {
                const response = await fetch(
                `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
                {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ contents: [{ parts: [{ text: prompt }] }] }),
                }
                );
                if (!response.ok) throw new Error('Erreur API');
                const data = await response.json();
                return data.candidates?.[0]?.content?.parts?.[0]?.text || "Impossible de générer le texte.";
            } catch (error) {
                console.error("Gemini API Error:", error);
                alert("Erreur lors de l'appel à l'IA. Vérifiez votre connexion.");
                return null;
            }
        };

        // --- COMPOSANTS UI ---
        const Button = ({ children, onClick, variant = 'primary', className = '', icon: Icon, disabled = false, title = '', isLoading = false }) => {
            const baseStyle = "px-4 py-2 rounded-lg font-medium transition-all duration-200 flex items-center gap-2 shadow-sm disabled:opacity-50 disabled:cursor-not-allowed";
            const variants = {
                primary: "bg-blue-600 text-white hover:bg-blue-700 active:transform active:scale-95",
                secondary: "bg-white text-gray-700 border border-gray-200 hover:bg-gray-50",
                danger: "bg-red-50 text-red-600 hover:bg-red-100 border border-red-200",
                ghost: "bg-transparent text-gray-600 hover:bg-gray-100 shadow-none",
                success: "bg-green-600 text-white hover:bg-green-700",
                warning: "bg-orange-100 text-orange-700 hover:bg-orange-200 border border-orange-200",
                ai: "bg-purple-600 text-white hover:bg-purple-700 border border-purple-200"
            };
            return (
                <button onClick={onClick} disabled={disabled || isLoading} title={title} className={`${baseStyle} ${variants[variant]} ${className}`}>
                    {isLoading ? <Loader2 size={18} className="animate-spin" /> : Icon && <Icon size={18} />}
                    {children}
                </button>
            );
        };

        const Card = ({ title, children, className = '', action }) => (
            <div className={`bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden ${className}`}>
                {title && (
                <div className="px-6 py-4 border-b border-gray-100 bg-gray-50 flex justify-between items-center">
                    <h3 className="font-semibold text-gray-800">{title}</h3>
                    {action}
                </div>
                )}
                <div className="p-6">{children}</div>
            </div>
        );

        const Input = ({ label, value, onChange, type = "text", placeholder = "", className = "" }) => (
            <div className={`mb-4 ${className}`}>
                {label && <label className="block text-sm font-medium text-gray-700 mb-1">{label}</label>}
                <input
                type={type}
                value={value}
                onChange={(e) => onChange(e.target.value)}
                className="w-full px-3 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-colors"
                placeholder={placeholder}
                />
            </div>
        );

        // --- DONNEES PAR DEFAUT ---
        const defaultPdfConfig = {
            primaryColor: '#000000', 
            secondaryColor: '#2563EB', 
            density: 'comfortable',
            showEmptyCompanies: true,
            footerAuthor: "L'Architecte / Le Maître d'Oeuvre",
            footerLegal: "Sans observations écrites à l'OPC dans les 8 jours suivant la réunion, le présent compte rendu est considéré comme accepté sans réserve.",
            sections: [
                { id: 'header', label: "En-tête & Info", visible: true, isCustom: false },
                { id: 'attendance', label: "Présences & Convocations", visible: true, isCustom: false },
                { id: 'general', label: "Généralités", visible: true, isCustom: false },
                { id: 'planning', label: "Planning Prévisionnel", visible: true, isCustom: false },
                { id: 'remarks', label: "Remarques", visible: true, isCustom: false },
                { id: 'choices', label: "Choix Validés", visible: true, isCustom: false },
                { id: 'photos', label: "Photos", visible: true, isCustom: false },
                { id: 'footer', label: "Pied de page", visible: true, isCustom: false },
            ]
        };

        const defaultProject = {
            id: 'default',
            name: 'Nouveau Chantier',
            logo: null,
            pdfConfig: defaultPdfConfig,
            customSections: [],
            crInfo: { 
                number: 1, 
                date: new Date().toISOString().split('T')[0],
                nextMeetingDate: '',
                importantInfo: "Chaque semaine, les nouvelles remarques sont indiquées en bleu et en gras."
            },
            companies: [
                { id: '01', role: 'ENT', lot: 'Menuiseries', name: 'ESPACE MENUISERIE', contact: 'Arnaud SIMON', presence: 'P', absences: 0, convocation: '11:00', isConvocated: true, details: 'arnaud@espacemenuiserie21.fr\n06 61 74 72 72' },
                { id: '02', role: 'ENT', lot: 'Plâtrerie', name: "WE SOL'D", contact: 'Nicolas NOUVELLON', presence: 'A', absences: 2, convocation: '11:00', isConvocated: true, details: 'wesold@wesold.fr\n06 33 67 08 90' },
                { id: '03', role: 'ENT', lot: 'Peinture', name: 'DELAGNEAU', contact: 'Damien BOURSIER', presence: 'P', absences: 1, convocation: '11:00', isConvocated: true, details: 'accueil@delagneau.fr\n06 89 33 17 39' },
                { id: '06', role: 'ENT', lot: 'Gros Oeuvre', name: 'GC BAT', contact: 'Antoine BENOIT', presence: 'P', absences: 0, convocation: '11:00', isConvocated: true, details: 'a.benoit@gcbat.fr\n07 87 32 07 21' },
                { id: 'MOA', role: 'MOA', lot: 'Client', name: 'ICB', contact: 'Direction', presence: 'P', absences: 0, convocation: '10:00', isConvocated: true, details: 'direction@icb-bourgogne.fr' },
                { id: 'MOE', role: 'MOE', lot: 'Archi', name: 'COSINUS', contact: 'Chef de Projet', presence: 'P', absences: 0, convocation: '10:00', isConvocated: true, details: 'contact@cosinus.fr' },
            ],
            generalities: [
                { id: 1, title: 'Organisation', content: 'Le port des EPI est obligatoire.' }
            ],
            delays: [],
            choices: [],
            photos: [],
            planning: [
                { id: 1, lotId: '06', task: "Modification fosse et caniveaux", dateStart: "2025-04-10", duration: "6", dateEnd: "2025-04-16", progress: 100 },
                { id: 2, lotId: '01', task: "Livraison huisseries", dateStart: "2025-04-20", duration: "5", dateEnd: "2025-04-25", progress: 100 },
            ],
            remarks: [
                { id: 1, type: 'ENT', lotId: '01', text: "Les claustras des salles d'attentes ne seront pas réalisés.", number: "1.1", dateDemand: "", dateDone: "", status: 'normal' },
                { id: 2, type: 'ENT', lotId: '03', text: "Prévoir un vide de 10cm derrière le caisson.", number: "7.1", dateDemand: "12/05/2025", dateDone: "", status: 'modified' },
                { id: 3, type: 'MOA', lotId: null, text: "Validation des plans électriques.", number: "7.2", dateDemand: "20/05/2025", dateDone: "", status: 'normal' }
            ]
        };

        // --- SOUS-COMPOSANTS ---
        const ConfigView = ({ project, updateProject, fileInputRef, handleLogoUpload }) => {
            const [newSectionTitle, setNewSectionTitle] = useState('');
            const [newParticipant, setNewParticipant] = useState({
                role: 'ENT', id: '', lot: '', name: '', contact: '', details: '', convocation: '11:00', isConvocated: true
            });
            const [editingId, setEditingId] = useState(null);

            const addOrUpdateParticipant = () => {
                if(!newParticipant.name) return;

                if (editingId) {
                    const updatedCompanies = project.companies.map(c => 
                        c.id === editingId ? { ...c, ...newParticipant } : c
                    );
                    updateProject({ companies: updatedCompanies });
                    setEditingId(null);
                } else {
                    const updatedCompanies = [...project.companies, { ...newParticipant, presence: 'P', absences: 0, isConvocated: true }];
                    updateProject({ companies: updatedCompanies });
                }
                setNewParticipant({ role: 'ENT', id: '', lot: '', name: '', contact: '', details: '', convocation: '11:00', isConvocated: true });
            };

            const startEditParticipant = (c) => {
                setNewParticipant(c);
                setEditingId(c.id);
            };

            const cancelEdit = () => {
                setNewParticipant({ role: 'ENT', id: '', lot: '', name: '', contact: '', details: '', convocation: '11:00', isConvocated: true });
                setEditingId(null);
            };

            const removeParticipant = (id) => {
                if(confirm("Supprimer cet intervenant ?")) {
                    updateProject({ companies: project.companies.filter(c => c.id !== id) });
                    if(editingId === id) cancelEdit();
                }
            };

            const addCustomSection = () => {
                if(!newSectionTitle) return;
                const newId = `custom_${Date.now()}`;
                
                const updatedCustomSections = [...(project.customSections || []), { id: newId, title: newSectionTitle, content: '' }];
                const newPdfSections = [...project.pdfConfig.sections];
                const footerIndex = newPdfSections.findIndex(s => s.id === 'footer');
                const insertIndex = footerIndex !== -1 ? footerIndex : newPdfSections.length;
                
                newPdfSections.splice(insertIndex, 0, { 
                    id: newId, 
                    label: `[Perso] ${newSectionTitle}`, 
                    visible: true, 
                    isCustom: true 
                });

                updateProject({ 
                    customSections: updatedCustomSections,
                    pdfConfig: { ...project.pdfConfig, sections: newPdfSections }
                });
                setNewSectionTitle('');
            };

            const removeCustomSection = (id) => {
                const updatedCustom = project.customSections.filter(s => s.id !== id);
                const updatedPdfSections = project.pdfConfig.sections.filter(s => s.id !== id);
                updateProject({
                    customSections: updatedCustom,
                    pdfConfig: { ...project.pdfConfig, sections: updatedPdfSections }
                });
            };

            return (
            <div className="space-y-6">
                <Card title="Annuaire des Intervenants (Entreprises, Clients, MOE...)">
                    <div className={`bg-gray-50 p-4 rounded-lg mb-4 grid grid-cols-1 md:grid-cols-7 gap-2 items-end border ${editingId ? 'border-orange-300 bg-orange-50' : 'border-gray-200'}`}>
                        <div className="md:col-span-1">
                            <label className="text-xs font-bold text-gray-500">Rôle</label>
                            <select className="w-full border rounded p-2 text-sm" value={newParticipant.role} onChange={(e) => setNewParticipant({...newParticipant, role: e.target.value})}>
                                <option value="ENT">Entreprise</option>
                                <option value="MOA">Maître d'Ouvrage</option>
                                <option value="MOE">Maître d'Oeuvre</option>
                                <option value="EXT">Intervenant Ext.</option>
                            </select>
                        </div>
                        <div className="md:col-span-1">
                            <label className="text-xs font-bold text-gray-500">ID / Code</label>
                            <input className="w-full border rounded p-2 text-sm" placeholder="Ex: 01" value={newParticipant.id} onChange={(e) => setNewParticipant({...newParticipant, id: e.target.value})} disabled={!!editingId} title={editingId ? "L'ID ne peut pas être modifié" : ""} />
                        </div>
                        <div className="md:col-span-1">
                            <label className="text-xs font-bold text-gray-500">Lot / Fonction</label>
                            <input className="w-full border rounded p-2 text-sm" placeholder="Ex: Plomberie" value={newParticipant.lot} onChange={(e) => setNewParticipant({...newParticipant, lot: e.target.value})} />
                        </div>
                        <div className="md:col-span-1">
                            <label className="text-xs font-bold text-gray-500">Société</label>
                            <input className="w-full border rounded p-2 text-sm" placeholder="Nom Sté" value={newParticipant.name} onChange={(e) => setNewParticipant({...newParticipant, name: e.target.value})} />
                        </div>
                        <div className="md:col-span-1">
                            <label className="text-xs font-bold text-gray-500">Contact</label>
                            <input className="w-full border rounded p-2 text-sm" placeholder="Nom Prénom" value={newParticipant.contact} onChange={(e) => setNewParticipant({...newParticipant, contact: e.target.value})} />
                        </div>
                        <div className="md:col-span-1">
                            <label className="text-xs font-bold text-gray-500">Coordonnées</label>
                            <input className="w-full border rounded p-2 text-sm" placeholder="Mail / Tel" value={newParticipant.details} onChange={(e) => setNewParticipant({...newParticipant, details: e.target.value})} />
                        </div>
                        <div className="md:col-span-1 flex gap-1">
                            {editingId && (
                                <Button onClick={cancelEdit} variant="secondary" className="w-full justify-center h-[38px] px-2" title="Annuler">
                                    <X size={16}/>
                                </Button>
                            )}
                            <Button onClick={addOrUpdateParticipant} icon={editingId ? RefreshCw : UserPlus} variant={editingId ? "warning" : "primary"} className="w-full justify-center h-[38px]">
                                {editingId ? 'Maj' : 'Ajouter'}
                            </Button>
                        </div>
                    </div>

                    <div className="overflow-x-auto">
                        <table className="w-full text-sm border-collapse">
                            <thead>
                                <tr className="bg-gray-100 text-left">
                                    <th className="p-2 border">Rôle</th>
                                    <th className="p-2 border">ID</th>
                                    <th className="p-2 border">Lot</th>
                                    <th className="p-2 border">Société</th>
                                    <th className="p-2 border">Contact</th>
                                    <th className="p-2 border w-20 text-center">Actions</th>
                                </tr>
                            </thead>
                            <tbody>
                                {project.companies.map(c => (
                                    <tr key={c.id + c.name} className={`border-b hover:bg-gray-50 ${editingId === c.id ? 'bg-orange-50' : ''}`}>
                                        <td className="p-2 border font-bold text-xs">
                                            <span className={`px-2 py-1 rounded ${c.role === 'MOA' ? 'bg-purple-100 text-purple-800' : c.role === 'MOE' ? 'bg-blue-100 text-blue-800' : c.role === 'EXT' ? 'bg-yellow-100 text-yellow-800' : 'bg-gray-100 text-gray-800'}`}>
                                                {c.role}
                                            </span>
                                        </td>
                                        <td className="p-2 border font-mono">{c.id}</td>
                                        <td className="p-2 border">{c.lot}</td>
                                        <td className="p-2 border font-bold">{c.name}</td>
                                        <td className="p-2 border">{c.contact}</td>
                                        <td className="p-2 border text-center">
                                            <div className="flex justify-center gap-2">
                                                <button onClick={() => startEditParticipant(c)} className="text-blue-500 hover:text-blue-700" title="Modifier"><PenSquare size={16}/></button>
                                                <button onClick={() => removeParticipant(c.id)} className="text-red-400 hover:text-red-600" title="Supprimer"><Trash2 size={16}/></button>
                                            </div>
                                        </td>
                                    </tr>
                                ))}
                            </tbody>
                        </table>
                    </div>
                </Card>

                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <Card title="Identité & Info">
                        <div className="flex items-center gap-4 mb-4">
                        <div className="w-24 h-24 bg-gray-100 rounded-lg flex items-center justify-center overflow-hidden border">
                            {project.logo ? <img src={project.logo} alt="Logo" className="w-full h-full object-contain" /> : <ImageIcon className="text-gray-400"/>}
                        </div>
                        <div>
                            <input type="file" ref={fileInputRef} onChange={handleLogoUpload} className="hidden" accept="image/*" />
                            <Button onClick={() => fileInputRef.current.click()} variant="secondary" icon={Camera}>Changer le logo</Button>
                        </div>
                        </div>
                        <label className="block text-sm font-medium text-gray-700 mb-1">Informations Importantes (Encart sous l'entête)</label>
                        <textarea 
                        className="w-full border rounded-lg p-2 text-sm" 
                        rows={3}
                        value={project.crInfo.importantInfo}
                        onChange={(e) => updateProject({ crInfo: {...project.crInfo, importantInfo: e.target.value} })}
                        />
                    </Card>
                    <Card title="Réunions">
                        <div className="flex gap-4 mb-4">
                        <div className="flex-1">
                            <Input label="Date Réunion Actuelle" type="date" value={project.crInfo.date} onChange={(v) => updateProject({ crInfo: {...project.crInfo, date: v} })} />
                        </div>
                        <div className="flex-1">
                            <Input label="Date Prochaine Réunion" type="date" value={project.crInfo.nextMeetingDate} onChange={(v) => updateProject({ crInfo: {...project.crInfo, nextMeetingDate: v} })} />
                        </div>
                        </div>
                        <div className="bg-blue-50 p-4 rounded-lg flex justify-between items-center">
                        <div>
                            <div className="text-sm text-blue-800 font-bold">CR NUMÉRO</div>
                            <div className="text-4xl font-black text-blue-600">{project.crInfo.number}</div>
                        </div>
                        <Button onClick={() => updateProject({ crInfo: {...project.crInfo, number: project.crInfo.number + 1} })} icon={Plus}>Nouveau CR (+1)</Button>
                        </div>
                    </Card>
                </div>
                
                <Card title="Pied de Page & Rédacteur">
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                        <div>
                            <label className="block text-sm font-medium text-gray-700 mb-1">Nom du Rédacteur</label>
                            <input className="w-full border rounded p-2" value={project.pdfConfig.footerAuthor} onChange={(e) => updateProject({ pdfConfig: { ...project.pdfConfig, footerAuthor: e.target.value } })} placeholder="Ex: L'Architecte" />
                        </div>
                        <div>
                            <label className="block text-sm font-medium text-gray-700 mb-1">Texte Légal (Bas de page)</label>
                            <textarea className="w-full border rounded p-2 text-xs" rows={2} value={project.pdfConfig.footerLegal} onChange={(e) => updateProject({ pdfConfig: { ...project.pdfConfig, footerLegal: e.target.value } })} />
                        </div>
                    </div>
                </Card>

                <Card title="Ajouter des Sections au Rapport">
                    <div className="flex gap-2 mb-4">
                        <Input className="flex-1 mb-0" placeholder="Titre de la nouvelle section (ex: Qualité, Sécurité...)" value={newSectionTitle} onChange={setNewSectionTitle} />
                        <Button onClick={addCustomSection} icon={Plus}>Ajouter</Button>
                    </div>
                    <div className="space-y-2">
                        {(project.customSections || []).map(section => (
                            <div key={section.id} className="p-3 border rounded bg-gray-50">
                                <div className="flex justify-between items-center mb-2">
                                <span className="font-bold">{section.title}</span>
                                <button onClick={() => removeCustomSection(section.id)} className="text-red-500 hover:text-red-700"><Trash2 size={16}/></button>
                                </div>
                                <textarea 
                                className="w-full border rounded p-2 text-sm" 
                                placeholder="Contenu de la section..."
                                rows={3}
                                value={section.content}
                                onChange={(e) => {
                                    const updated = project.customSections.map(s => s.id === section.id ? {...s, content: e.target.value} : s);
                                    updateProject({ customSections: updated });
                                }}
                                />
                            </div>
                        ))}
                    </div>
                </Card>

                <Card title="Mise en Page et Couleurs">
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
                        <div>
                            <h4 className="font-bold text-gray-700 mb-4 flex items-center gap-2"><Palette size={16}/> Style</h4>
                            <div className="space-y-4">
                                <div>
                                    <label className="block text-sm text-gray-600 mb-1">Couleur Principale (Titres)</label>
                                    <div className="flex items-center gap-2">
                                        <input type="color" value={project.pdfConfig.primaryColor} onChange={(e) => updateProject({ pdfConfig: { ...project.pdfConfig, primaryColor: e.target.value } })} className="h-8 w-16 cursor-pointer border rounded" />
                                        <span className="text-xs text-gray-500">{project.pdfConfig.primaryColor}</span>
                                    </div>
                                </div>
                                <div>
                                    <label className="block text-sm text-gray-600 mb-1">Couleur "Nouvelles Remarques" (Bleu)</label>
                                    <div className="flex items-center gap-2">
                                        <input type="color" value={project.pdfConfig.secondaryColor} onChange={(e) => updateProject({ pdfConfig: { ...project.pdfConfig, secondaryColor: e.target.value } })} className="h-8 w-16 cursor-pointer border rounded" />
                                        <span className="text-xs text-gray-500">{project.pdfConfig.secondaryColor}</span>
                                    </div>
                                </div>
                                <div className="flex items-center gap-2 mt-4">
                                    <input 
                                        type="checkbox" 
                                        id="showEmpty"
                                        checked={project.pdfConfig.showEmptyCompanies} 
                                        onChange={(e) => updateProject({ pdfConfig: { ...project.pdfConfig, showEmptyCompanies: e.target.checked } })}
                                        className="w-4 h-4 text-blue-600 rounded"
                                    />
                                    <label htmlFor="showEmpty" className="text-sm font-medium text-gray-700">Afficher les entreprises sans remarques</label>
                                </div>
                            </div>
                        </div>
                        <div>
                            <h4 className="font-bold text-gray-700 mb-4 flex items-center gap-2"><LayoutTemplate size={16}/> Ordre des Sections</h4>
                            <div className="space-y-2">
                                {project.pdfConfig.sections.map((section, index) => (
                                    <div key={section.id} className="flex items-center justify-between p-2 bg-gray-50 border rounded-lg hover:bg-gray-100 transition-colors">
                                        <div className="flex items-center gap-3">
                                            <button onClick={() => {
                                                const sections = project.pdfConfig.sections.map(s => s.id === section.id ? { ...s, visible: !s.visible } : s);
                                                updateProject({ pdfConfig: { ...project.pdfConfig, sections } });
                                            }} className={`text-gray-500 hover:text-blue-600 ${!section.visible && 'opacity-30'}`}>
                                                {section.visible ? <Eye size={16}/> : <EyeOff size={16}/>}
                                            </button>
                                            <span className={`text-sm font-medium ${!section.visible && 'text-gray-400 line-through'}`}>{section.label}</span>
                                        </div>
                                        <div className="flex gap-1">
                                            <button onClick={() => {
                                                if (index === 0) return;
                                                const sections = [...project.pdfConfig.sections];
                                                [sections[index], sections[index - 1]] = [sections[index - 1], sections[index]];
                                                updateProject({ pdfConfig: { ...project.pdfConfig, sections } });
                                            }} disabled={index === 0} className="p-1 text-gray-400 hover:text-black disabled:opacity-20"><MoveUp size={14}/></button>
                                            <button onClick={() => {
                                                if (index === project.pdfConfig.sections.length - 1) return;
                                                const sections = [...project.pdfConfig.sections];
                                                [sections[index], sections[index + 1]] = [sections[index + 1], sections[index]];
                                                updateProject({ pdfConfig: { ...project.pdfConfig, sections } });
                                            }} disabled={index === project.pdfConfig.sections.length - 1} className="p-1 text-gray-400 hover:text-black disabled:opacity-20"><MoveDown size={14}/></button>
                                        </div>
                                    </div>
                                ))}
                            </div>
                        </div>
                    </div>
                </Card>
            </div>
            )};

        const AttendanceView = ({ project, togglePresence, updateProject }) => {
            const toggleConvocation = (c) => {
                const updatedCompanies = project.companies.map(comp => {
                    if (comp.id === c.id) {
                        return { ...comp, isConvocated: !comp.isConvocated };
                    }
                    return comp;
                });
                updateProject({ companies: updatedCompanies });
            };

            return (
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
                {project.companies.map(c => (
                <div key={c.id} className={`bg-white rounded-xl border shadow-sm p-4 flex flex-col justify-between transition-all ${!c.isConvocated ? 'opacity-90 border-gray-200 bg-gray-50' : 'border-blue-100'}`}>
                    <div>
                    <div className="flex justify-between items-start mb-2">
                        <span className={`font-mono text-xs font-bold px-2 py-1 rounded ${c.role === 'ENT' ? 'bg-gray-100' : 'bg-blue-100 text-blue-800'}`}>
                            {c.id} - {c.lot}
                        </span>
                        <div className="flex gap-2">
                            <div className="text-xs font-bold text-red-500 bg-red-50 px-2 py-1 rounded-full">{c.absences} abs.</div>
                        </div>
                    </div>
                    <h3 className="font-bold text-gray-900 leading-tight">{c.name}</h3>
                    <p className="text-sm text-gray-500 mb-3">{c.contact}</p>
                    </div>
                    
                    <div className="mb-4">
                        <h4 className="text-xs font-bold text-gray-400 uppercase mb-1">Présence Réunion</h4>
                        <div className="grid grid-cols-3 gap-2">
                            <button onClick={() => togglePresence(c.id, 'P')} className={`py-2 rounded-lg text-sm font-bold transition-colors ${c.presence === 'P' ? 'bg-green-600 text-white' : 'bg-gray-100 text-gray-400 hover:bg-gray-200'}`}>Présent</button>
                            <button onClick={() => togglePresence(c.id, 'A')} className={`py-2 rounded-lg text-sm font-bold transition-colors ${c.presence === 'A' ? 'bg-red-600 text-white' : 'bg-gray-100 text-gray-400 hover:bg-gray-200'}`}>Absent</button>
                            <button onClick={() => togglePresence(c.id, 'Ex')} className={`py-2 rounded-lg text-sm font-bold transition-colors ${c.presence === 'Ex' ? 'bg-orange-500 text-white' : 'bg-gray-100 text-gray-400 hover:bg-gray-200'}`}>Excusé</button>
                        </div>
                    </div>

                    <div className="pt-3 border-t">
                        <h4 className="text-xs font-bold text-gray-400 uppercase mb-2 flex items-center justify-between">
                            <span>Convocation Prochaine</span>
                            <button 
                                onClick={() => toggleConvocation(c)}
                                className={`text-[10px] px-2 py-0.5 rounded border ${c.isConvocated ? 'bg-blue-50 text-blue-600 border-blue-200' : 'bg-gray-100 text-gray-500'}`}
                            >
                                {c.isConvocated ? 'Convoqué' : 'Non Convoqué'}
                            </button>
                        </h4>
                        
                        {c.isConvocated ? (
                            <div className="flex items-center gap-2">
                                <Clock size={16} className="text-blue-500"/>
                                <span className="text-xs text-gray-600">Heure :</span>
                                <input 
                                    type="time" 
                                    className="text-xs border rounded px-1 w-20 text-center font-bold text-gray-900 bg-white" 
                                    value={c.convocation} 
                                    onChange={(e) => {
                                        const updated = project.companies.map(co => co.id === c.id ? {...co, convocation: e.target.value} : co);
                                        updateProject({ companies: updated });
                                    }}
                                />
                            </div>
                        ) : (
                            <div className="text-xs text-gray-400 italic py-1 flex items-center gap-2">
                                <BellOff size={14}/> Pas de convocation prévue.
                            </div>
                        )}
                    </div>
                </div>
                ))}
            </div>
            )};

        const GeneralitiesView = ({ project, updateProject }) => {
            const [newGeneral, setNewGeneral] = useState({ title: '', content: '' });
            const [newDelay, setNewDelay] = useState({ date: '', type: 'Intempérie', reason: '' });
            const [isGenerating, setIsGenerating] = useState(false);

            const generateSummary = async () => {
                setIsGenerating(true);
                const context = `Retards en cours: ${JSON.stringify(project.delays)}. Remarques récentes: ${JSON.stringify(project.remarks.slice(-5))}.`;
                const prompt = `Tu es un architecte. Rédige un paragraphe court et professionnel pour la section "Généralités" d'un compte-rendu de chantier. Résume la situation basée sur ces données : ${context}. Commence directement par le texte.`;
                
                const result = await callGemini(prompt);
                if (result) {
                setNewGeneral({ title: 'Synthèse Avancement', content: result });
                }
                setIsGenerating(false);
            };

            const updateGeneral = (id, field, value) => {
                const updated = project.generalities.map(g => g.id === id ? { ...g, [field]: value } : g);
                updateProject({ generalities: updated });
            };

            return (
                <div className="space-y-6">
                <Card title="Généralités & Remarques Communes">
                    <div className="bg-purple-50 p-4 rounded-lg mb-4 border border-purple-100">
                        <div className="flex justify-between items-center mb-2">
                        <h4 className="text-sm font-bold text-purple-800 flex items-center gap-2"><Sparkles size={16}/> Assistant IA</h4>
                        </div>
                        <p className="text-xs text-purple-600 mb-3">Générez automatiquement une synthèse de l'avancement basée sur les retards et les dernières remarques.</p>
                        <Button variant="ai" onClick={generateSummary} isLoading={isGenerating} icon={Sparkles} className="w-full justify-center text-sm py-1">Générer une synthèse</Button>
                    </div>

                    <div className="flex gap-2 mb-4">
                        <Input className="flex-1 mb-0" placeholder="Titre (ex: Sécurité)" value={newGeneral.title} onChange={(v) => setNewGeneral({...newGeneral, title: v})} />
                        <Input className="flex-[2] mb-0" placeholder="Contenu..." value={newGeneral.content} onChange={(v) => setNewGeneral({...newGeneral, content: v})} />
                        <Button onClick={() => {
                        if(newGeneral.title) {
                            updateProject({ generalities: [...project.generalities, { id: Date.now(), ...newGeneral }] });
                            setNewGeneral({ title: '', content: '' });
                        }
                        }} icon={Plus}>Ajouter</Button>
                    </div>
                    <div className="space-y-2">
                        {project.generalities.map(g => (
                        <div key={g.id} className="p-3 border rounded bg-gray-50 flex flex-col gap-2">
                            <div className="flex justify-between items-center">
                                <input 
                                    className="font-bold text-blue-900 bg-transparent border-b border-transparent hover:border-gray-300 focus:border-blue-500 outline-none w-full"
                                    value={g.title}
                                    onChange={(e) => updateGeneral(g.id, 'title', e.target.value)}
                                />
                                <button onClick={() => updateProject({ generalities: project.generalities.filter(x => x.id !== g.id) })} className="text-red-400 ml-2"><Trash2 size={16}/></button>
                            </div>
                            <textarea 
                                className="text-gray-700 bg-transparent border border-transparent hover:border-gray-300 focus:border-blue-500 outline-none w-full text-sm resize-y"
                                value={g.content}
                                rows={2}
                                onChange={(e) => updateGeneral(g.id, 'content', e.target.value)}
                            />
                        </div>
                        ))}
                    </div>
                </Card>
                
                <Card title="Retards & Intempéries">
                    <div className="flex gap-2 mb-4">
                        <Input type="date" className="w-32 mb-0" value={newDelay.date} onChange={(v) => setNewDelay({...newDelay, date: v})} />
                        <select className="border rounded px-2" value={newDelay.type} onChange={(e) => setNewDelay({...newDelay, type: e.target.value})}>
                            <option>Intempérie</option>
                            <option>Retard Ent.</option>
                            <option>Autre</option>
                        </select>
                        <Input className="flex-1 mb-0" placeholder="Raison..." value={newDelay.reason} onChange={(v) => setNewDelay({...newDelay, reason: v})} />
                        <Button onClick={() => {
                            updateProject({ delays: [...project.delays, { id: Date.now(), ...newDelay }] });
                        }} icon={Plus} />
                    </div>
                    <ul className="space-y-2">
                        {project.delays.map(d => (
                            <li key={d.id} className="text-sm flex justify-between border-b pb-1">
                                <span><strong>{d.date}</strong> ({d.type}) : {d.reason}</span>
                                <button className="text-red-400" onClick={() => updateProject({ delays: project.delays.filter(x => x.id !== d.id) })}><Trash2 size={14}/></button>
                            </li>
                        ))}
                    </ul>
                </Card>
                </div>
            );
        };

        const PlanningView = ({ project, updateProject }) => {
            const [newTask, setNewTask] = useState({ task: '', lotId: '', dateStart: '', duration: '', dateEnd: '', progress: 0 });

            const addTask = () => {
                if (!newTask.task) return;
                updateProject({ planning: [...project.planning, { id: Date.now(), ...newTask }] });
                setNewTask({ task: '', lotId: '', dateStart: '', duration: '', dateEnd: '', progress: 0 });
            };

            return (
                <div className="space-y-6">
                    <Card title="Planning Prévisionnel">
                        <div className="bg-gray-50 p-4 rounded-lg mb-6 grid grid-cols-1 md:grid-cols-6 gap-2 items-end">
                            <div className="md:col-span-1">
                                <label className="block text-xs font-bold text-gray-500 mb-1">Responsable</label>
                                <select className="w-full border rounded p-2 text-sm" value={newTask.lotId} onChange={(e) => setNewTask({...newTask, lotId: e.target.value})}>
                                    <option value="">Tous Corps d'Etat</option>
                                    {project.companies.map(c => <option key={c.id} value={c.id}>{c.name}</option>)}
                                </select>
                            </div>
                            <div className="md:col-span-2">
                                <label className="block text-xs font-bold text-gray-500 mb-1">Tâche</label>
                                <input className="w-full border rounded p-2 text-sm" placeholder="Description..." value={newTask.task} onChange={(e) => setNewTask({...newTask, task: e.target.value})} />
                            </div>
                            <div className="md:col-span-1">
                                <label className="block text-xs font-bold text-gray-500 mb-1">Début</label>
                                <input type="date" className="w-full border rounded p-2 text-sm" value={newTask.dateStart} onChange={(e) => setNewTask({...newTask, dateStart: e.target.value})} />
                            </div>
                            <div className="md:col-span-1">
                                <label className="block text-xs font-bold text-gray-500 mb-1">Durée (j)</label>
                                <input type="number" className="w-full border rounded p-2 text-sm" value={newTask.duration} onChange={(e) => setNewTask({...newTask, duration: e.target.value})} />
                            </div>
                            <div className="md:col-span-1">
                                <Button onClick={addTask} icon={Plus} className="w-full justify-center">Ajouter</Button>
                            </div>
                        </div>

                        <div className="space-y-2">
                            {project.planning.map(p => (
                                <div key={p.id} className="flex items-center gap-4 p-3 border rounded-lg hover:bg-gray-50">
                                    <div className="w-16 font-bold text-xs bg-gray-200 text-center py-1 rounded">{p.lotId || 'TCE'}</div>
                                    <div className="flex-1">
                                        <div className="font-medium text-sm">{p.task}</div>
                                        <div className="text-xs text-gray-500">
                                            Début: {p.dateStart || '-'} | Durée: {p.duration ? p.duration + 'j' : '-'} | Fin: {p.dateEnd || '-'}
                                        </div>
                                    </div>
                                    <div className="w-32 flex items-center gap-2">
                                        <input type="range" min="0" max="100" className="w-full" value={p.progress} onChange={(e) => {
                                            const updated = project.planning.map(x => x.id === p.id ? {...x, progress: parseInt(e.target.value)} : x);
                                            updateProject({ planning: updated });
                                        }}/>
                                        <span className="text-xs font-bold w-8">{p.progress}%</span>
                                    </div>
                                    <button className="text-red-400 hover:text-red-600" onClick={() => updateProject({ planning: project.planning.filter(x => x.id !== p.id) })}><Trash2 size={16}/></button>
                                </div>
                            ))}
                        </div>
                    </Card>
                </div>
            );
        };

        const ChoicesView = ({ project, updateProject }) => {
            const [newChoice, setNewChoice] = useState({ lotId: '', description: '' });

            return (
                <Card title="Choix Clients (Matériaux/Couleurs)">
                    <div className="flex gap-2 mb-4">
                    <select className="border rounded px-2 w-1/3" value={newChoice.lotId} onChange={(e) => setNewChoice({...newChoice, lotId: e.target.value})}>
                        <option value="">Concernant...</option>
                        {project.companies.map(c => <option key={c.id} value={c.id}>{c.lot} ({c.name})</option>)}
                    </select>
                    <Input className="flex-1 mb-0" placeholder="Choix (ex: Carrelage Gris)" value={newChoice.description} onChange={(v) => setNewChoice({...newChoice, description: v})} />
                    <Button onClick={() => {
                        if(newChoice.description) {
                            updateProject({ choices: [...project.choices, { id: Date.now(), ...newChoice }] });
                            setNewChoice({ lotId: '', description: '' });
                        }
                    }} icon={Plus} />
                    </div>
                    <ul className="space-y-2">
                    {project.choices.map(c => (
                        <li key={c.id} className="text-sm flex justify-between border-b pb-1">
                            <span><span className="font-bold">{c.lotId ? `${c.lotId} : ` : ''}</span>{c.description}</span>
                            <button className="text-red-400" onClick={() => updateProject({ choices: project.choices.filter(x => x.id !== c.id) })}><Trash2 size={14}/></button>
                        </li>
                    ))}
                    </ul>
                </Card>
            );
        };

        const RemarksView = ({ project, updateProject, addRemarkParent }) => {
            const [filterType, setFilterType] = useState('ALL');
            const [newRemark, setNewRemark] = useState({ lotId: '', text: '', dateDemand: '', type: 'ENT' });
            const [isImproving, setIsImproving] = useState(false);
            
            // États pour l'édition d'une remarque
            const [editingId, setEditingId] = useState(null);
            const [tempRemark, setTempRemark] = useState({});

            const categories = [
            { id: 'MOA', label: "Maîtrise d'Ouvrage" },
            { id: 'MOE', label: "Maîtrise d'Oeuvre" },
            { id: 'EXT', label: "Intervenants Ext." },
            { id: 'ENT', label: "Entreprises" }
            ];

            const filteredRemarks = project.remarks.filter(r => filterType === 'ALL' || (filterType === 'ENT' ? r.type === 'ENT' : r.type === filterType));

            const handleAdd = () => {
                addRemarkParent(newRemark);
                setNewRemark({ ...newRemark, text: '', dateDemand: '' });
            };

            const improveRemark = async () => {
                if (!newRemark.text) return;
                setIsImproving(true);
                const prompt = `Tu es un assistant pour un maître d'oeuvre. Reformule cette remarque de chantier de manière professionnelle, précise et contractuelle, en une seule phrase : "${newRemark.text}"`;
                const result = await callGemini(prompt);
                if (result) {
                    setNewRemark(prev => ({ ...prev, text: result }));
                }
                setIsImproving(false);
            };

            // --- LOGIQUE EDITION ---
            const startEdit = (remark) => {
                setEditingId(remark.id);
                setTempRemark({ ...remark });
            };

            const cancelEdit = () => {
                setEditingId(null);
                setTempRemark({});
            };

            const saveEdit = () => {
                const updatedRemarks = project.remarks.map(r => 
                    r.id === editingId ? { ...r, ...tempRemark } : r
                );
                updateProject({ remarks: updatedRemarks });
                setEditingId(null);
                setTempRemark({});
            };

            const isNewRemark = (rem) => {
                if (rem.status === 'modified') return true;
                const prefix = rem.number.split('.')[0];
                return parseInt(prefix) === parseInt(project.crInfo.number);
            };

            return (
            <div className="space-y-6">
                <Card title="Saisie des Remarques">
                    <div className="grid grid-cols-1 md:grid-cols-6 gap-2 mb-4 bg-gray-50 p-4 rounded-lg items-end">
                        <div className="md:col-span-1">
                        <label className="text-xs font-bold text-gray-500">Type</label>
                        <select className="w-full border rounded p-2" value={newRemark.type} onChange={(e) => setNewRemark({...newRemark, type: e.target.value})}>
                            {categories.map(c => <option key={c.id} value={c.id}>{c.label}</option>)}
                        </select>
                        </div>
                        <div className="md:col-span-1">
                        <label className="text-xs font-bold text-gray-500">Intervenant</label>
                        <select className="w-full border rounded p-2" value={newRemark.lotId} onChange={(e) => setNewRemark({...newRemark, lotId: e.target.value})}>
                            <option value="">Sélectionner...</option>
                            {project.companies.filter(c => c.role === newRemark.type).map(c => (
                                <option key={c.id} value={c.id}>{c.id} - {c.lot} ({c.name})</option>
                            ))}
                            <optgroup label="Autres">
                                {project.companies.filter(c => c.role !== newRemark.type).map(c => (
                                    <option key={c.id} value={c.id}>{c.id} - {c.lot} ({c.name})</option>
                                ))}
                            </optgroup>
                        </select>
                        </div>
                        <div className="md:col-span-2 relative">
                        <label className="text-xs font-bold text-gray-500">Remarque</label>
                        <div className="flex gap-2">
                            <input className="w-full border rounded p-2 pr-10" placeholder="Texte..." value={newRemark.text} onChange={(e) => setNewRemark({...newRemark, text: e.target.value})} />
                            <Button variant="ai" onClick={improveRemark} isLoading={isImproving} icon={Sparkles} title="Améliorer la rédaction avec l'IA" className="p-2 h-[42px] aspect-square flex justify-center items-center" />
                        </div>
                        </div>
                        <div className="md:col-span-1">
                        <label className="text-xs font-bold text-gray-500">Pour le</label>
                        <input type="date" className="w-full border rounded p-2" value={newRemark.dateDemand} onChange={(e) => setNewRemark({...newRemark, dateDemand: e.target.value})} />
                        </div>
                        <div className="md:col-span-1">
                        <Button onClick={handleAdd} icon={Plus} className="w-full justify-center">Ajouter</Button>
                        </div>
                    </div>

                    <div className="flex gap-2 mb-4 overflow-x-auto pb-2">
                        <button onClick={() => setFilterType('ALL')} className={`px-3 py-1 rounded-full text-xs font-bold ${filterType === 'ALL' ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}>TOUT</button>
                        {categories.map(c => (
                        <button key={c.id} onClick={() => setFilterType(c.id)} className={`px-3 py-1 rounded-full text-xs font-bold ${filterType === c.id ? 'bg-blue-600 text-white' : 'bg-gray-200'}`}>{c.label}</button>
                        ))}
                    </div>

                    <div className="space-y-2">
                        {filteredRemarks.map(r => {
                        // MODE ÉDITION
                        if (editingId === r.id) {
                            return (
                                <div key={r.id} className="p-3 border-2 border-orange-300 bg-orange-50 rounded-lg flex flex-col gap-3">
                                    <div className="flex gap-2">
                                        <div className="flex-1">
                                            <label className="text-[10px] font-bold text-gray-500 uppercase">Type</label>
                                            <select 
                                                    className="w-full border rounded p-1 text-sm" 
                                                    value={tempRemark.type} 
                                                    onChange={(e) => setTempRemark({...tempRemark, type: e.target.value})}
                                            >
                                                {categories.map(c => <option key={c.id} value={c.id}>{c.label}</option>)}
                                            </select>
                                        </div>
                                        <div className="flex-1">
                                            <label className="text-[10px] font-bold text-gray-500 uppercase">Intervenant</label>
                                            <select 
                                                    className="w-full border rounded p-1 text-sm" 
                                                    value={tempRemark.lotId} 
                                                    onChange={(e) => setTempRemark({...tempRemark, lotId: e.target.value})}
                                            >
                                                {project.companies.map(c => <option key={c.id} value={c.id}>{c.id} - {c.lot} ({c.name})</option>)}
                                            </select>
                                        </div>
                                    </div>
                                    <div>
                                        <label className="text-[10px] font-bold text-gray-500 uppercase">Texte</label>
                                        <textarea 
                                                className="w-full border rounded p-2 text-sm" 
                                                rows={3} 
                                                value={tempRemark.text} 
                                                onChange={(e) => setTempRemark({...tempRemark, text: e.target.value})}
                                        />
                                    </div>
                                    <div className="flex justify-end gap-2">
                                        <Button onClick={cancelEdit} variant="secondary" className="h-8 text-sm">Annuler</Button>
                                        <Button onClick={saveEdit} variant="success" icon={CheckCircle2} className="h-8 text-sm">Enregistrer</Button>
                                    </div>
                                </div>
                            );
                        }

                        // MODE AFFICHAGE NORMAL
                        const isNew = isNewRemark(r);
                        const isMemo = r.status === 'memo';
                        return (
                            <div key={r.id} className={`p-3 border rounded-lg flex gap-4 items-start ${isMemo ? 'bg-gray-100 italic text-gray-500' : 'bg-white'}`}>
                                <div className={`font-mono font-bold w-12 ${isNew ? 'text-blue-600' : 'text-black'}`}>{r.number}</div>
                                <div className="flex-1">
                                    <div className="flex items-center gap-2 mb-1">
                                        {r.lotId && <span className="text-[10px] bg-gray-200 px-1 rounded font-bold">{r.lotId}</span>}
                                        <span className={`text-sm ${isNew ? 'text-blue-700 font-bold' : ''}`}>{r.text}</span>
                                    </div>
                                    <div className="flex gap-4 text-xs">
                                        <input type="date" className="border rounded px-1" title="Demandé pour" value={r.dateDemand} onChange={(e) => {
                                        updateProject({ remarks: project.remarks.map(x => x.id === r.id ? {...x, dateDemand: e.target.value} : x) });
                                        }} />
                                        <input type="date" className="border rounded px-1" title="Fait le" value={r.dateDone} onChange={(e) => {
                                        updateProject({ remarks: project.remarks.map(x => x.id === r.id ? {...x, dateDone: e.target.value} : x) });
                                        }} />
                                    </div>
                                </div>
                                <div className="flex flex-col gap-1">
                                    <button 
                                        onClick={() => startEdit(r)} 
                                        className="p-1 rounded text-blue-400 hover:text-blue-600 hover:bg-blue-50" 
                                        title="Modifier la remarque"
                                    >
                                        <PenSquare size={16}/>
                                    </button>
                                    <button onClick={() => updateProject({ remarks: project.remarks.map(x => x.id === r.id ? {...x, status: x.status === 'modified' ? 'normal' : 'modified'} : x) })} className={`p-1 rounded ${r.status === 'modified' ? 'bg-blue-100 text-blue-600' : 'text-gray-300 hover:bg-gray-100'}`}><Edit3 size={16}/></button>
                                    <button onClick={() => updateProject({ remarks: project.remarks.map(x => x.id === r.id ? {...x, status: x.status === 'memo' ? 'normal' : 'memo'} : x) })} className={`p-1 rounded ${r.status === 'memo' ? 'bg-gray-200 text-gray-600' : 'text-gray-300 hover:bg-gray-100'}`}><Archive size={16}/></button>
                                    <button onClick={() => updateProject({ remarks: project.remarks.filter(x => x.id !== r.id) })} className="p-1 rounded text-red-300 hover:text-red-500 hover:bg-red-50"><Trash2 size={16}/></button>
                                </div>
                            </div>
                        );
                        })}
                    </div>
                </Card>
            </div>
            );
        };

        const PhotosView = ({ project, updateProject, handlePhotoUpload, photoInputRef }) => (
            <Card title="Galerie Photos Avancement">
            <div className="mb-4">
                <input type="file" ref={photoInputRef} onChange={handlePhotoUpload} className="hidden" accept="image/*" />
                <Button onClick={() => photoInputRef.current.click()} icon={Camera}>Ajouter une photo</Button>
            </div>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
                {project.photos.map(p => (
                    <div key={p.id} className="relative group border rounded-lg overflow-hidden">
                        <img src={p.url} alt="Chantier" className="w-full h-32 object-cover" />
                        <input 
                        className="w-full p-1 text-xs border-t" 
                        placeholder="Légende..." 
                        value={p.caption} 
                        onChange={(e) => {
                            const updated = project.photos.map(ph => ph.id === p.id ? {...ph, caption: e.target.value} : ph);
                            updateProject({ photos: updated });
                        }}
                        />
                        <button 
                        onClick={() => updateProject({ photos: project.photos.filter(ph => ph.id !== p.id) })}
                        className="absolute top-1 right-1 bg-red-500 text-white p-1 rounded-full opacity-0 group-hover:opacity-100 transition-opacity"
                        >
                        <Trash2 size={12}/>
                        </button>
                    </div>
                ))}
            </div>
            </Card>
        );

        // --- SECTION PRINT PREVIEW COMPLÈTE (Version HTML) ---
        const PrintPreview = ({ project }) => {
            const { primaryColor, secondaryColor, density, sections, showEmptyCompanies, footerAuthor, footerLegal } = project.pdfConfig;
            const densityClass = density === 'compact' ? 'p-1 text-[9px]' : 'p-2 text-[10px]';
            const cellClass = `border border-black ${density === 'compact' ? 'px-1 py-0.5' : 'p-1'}`;

            const RemarkTable = ({ remarks }) => (
                <table className="w-full border-collapse border border-gray-400 text-[9px] table-fixed">
                <thead>
                    <tr className="bg-gray-100">
                        <th className="border border-gray-400 p-1 w-[5%] text-center">N°</th>
                        <th className="border border-gray-400 p-1 w-[79%] text-left">Libellé</th>
                        <th className="border border-gray-400 p-1 w-[8%] text-center">Pour le</th>
                        <th className="border border-gray-400 p-1 w-[8%] text-center">Fait le</th>
                    </tr>
                </thead>
                <tbody>
                    {remarks.map(r => {
                        const isNew = r.status === 'modified' || parseInt(r.number.split('.')[0]) === parseInt(project.crInfo.number);
                        const isMemo = r.status === 'memo';
                        const textColor = isNew ? secondaryColor : isMemo ? '#9ca3af' : 'black';
                        const fontWeight = isNew ? 'bold' : 'normal';
                        
                        return (
                            <tr key={r.id}>
                            <td className={`border border-gray-400 p-1 text-center align-top font-bold`} style={{ color: textColor }}>{r.number}</td>
                            <td className={`border border-gray-400 p-1 align-top whitespace-pre-wrap ${isMemo ? 'italic' : ''}`} style={{ color: textColor, fontWeight }}>{r.text}</td>
                            <td className="border border-gray-400 p-1 text-center align-top text-[8px]">{r.dateDemand ? new Date(r.dateDemand).toLocaleDateString() : ''}</td>
                            <td className="border border-gray-400 p-1 text-center align-top text-[8px]">{r.dateDone ? new Date(r.dateDone).toLocaleDateString() : ''}</td>
                            </tr>
                        )
                    })}
                </tbody>
                </table>
            );

            const renderSection = (sectionId) => {
                switch(sectionId) {
                    case 'header':
                        return (
                            <div className="mb-4">
                                <div className="flex justify-between items-start mb-2 border-b-2 pb-2" style={{ borderColor: primaryColor }}>
                                    <div className="w-1/4">
                                        {project.logo ? <img src={project.logo} alt="Logo" className="max-h-16 max-w-full object-contain" /> : <div className="font-bold text-lg text-gray-400">LOGO</div>}
                                    </div>
                                    <div className="w-2/4 text-center">
                                        <h1 className="text-xl font-black uppercase mb-1" style={{ color: primaryColor }}>{project.name}</h1>
                                        <div className="inline-block border-2 px-4 py-1 text-lg font-bold bg-gray-100" style={{ borderColor: primaryColor }}>
                                            COMPTE RENDU N° {project.crInfo.number}
                                        </div>
                                        <div className="mt-2 text-sm font-bold">
                                            Date : {new Date(project.crInfo.date).toLocaleDateString('fr-FR', { weekday: 'long', year: 'numeric', month: 'long', day: 'numeric' })}
                                        </div>
                                    </div>
                                    <div className="w-1/4 text-right">
                                        <div className="text-[10px]">Prochaine réunion :</div>
                                        <div className="font-bold text-sm text-red-600">
                                            {project.crInfo.nextMeetingDate 
                                            ? new Date(project.crInfo.nextMeetingDate).toLocaleDateString() 
                                            : 'À définir'}
                                        </div>
                                    </div>
                                </div>
                                {project.crInfo.importantInfo && (
                                    <div className="w-full mt-2 border border-red-500 bg-red-50 p-2 text-center text-red-700 font-bold mb-4 rounded text-xs">
                                        {project.crInfo.importantInfo}
                                    </div>
                                )}
                            </div>
                        );
                    case 'attendance':
                        return (
                            <div className="mb-6">
                                <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-1" style={{ backgroundColor: primaryColor }}>Présences & Convocations</h2>
                                <table className={`w-full border-collapse border border-black ${density === 'compact' ? 'text-[8px]' : 'text-[9px]'}`}>
                                    <thead>
                                        <tr className="bg-gray-200">
                                            <th className={`${cellClass} text-left w-1/4`}>Intervenant / Lot</th>
                                            <th className={`${cellClass} text-left w-1/4`}>Société / Contact</th>
                                            <th className={`${cellClass} text-center w-6`}>P</th>
                                            <th className={`${cellClass} text-center w-6`}>A</th>
                                            <th className={`${cellClass} text-center w-6`}>Ex</th>
                                            <th className={`${cellClass} text-center w-6`}>Abs.</th>
                                            <th className={`${cellClass} text-center w-12 text-red-600`}>Convoc</th>
                                            <th className={`${cellClass} text-left`}>Coordonnées</th>
                                        </tr>
                                    </thead>
                                    <tbody>
                                        <tr className="bg-gray-200"><td colSpan="8" className="border border-black font-bold text-center py-1 uppercase tracking-wider">ENTREPRISES</td></tr>
                                        {project.companies.filter(c => c.role === 'ENT').map(c => {
                                            const showPresence = c.isConvocated !== false;
                                            return (
                                                <tr key={c.id}>
                                                    <td className={`${cellClass} font-bold`}>{c.id} - {c.lot}</td>
                                                    <td className={cellClass}><span className="font-bold">{c.name}</span><br/>{c.contact}</td>
                                                    <td className={`${cellClass} text-center font-bold`}>{showPresence && c.presence === 'P' ? 'X' : ''}</td>
                                                    <td className={`${cellClass} text-center font-bold text-red-600`}>{showPresence && c.presence === 'A' ? 'X' : ''}</td>
                                                    <td className={`${cellClass} text-center`}>{showPresence && c.presence === 'Ex' ? 'X' : ''}</td>
                                                    <td className={`${cellClass} text-center bg-gray-100`}>{c.absences}</td>
                                                    <td className={`${cellClass} text-center font-bold text-red-600`}>{showPresence ? c.convocation : '-'}</td>
                                                    <td className={`${cellClass} whitespace-pre-line`}>{c.details}</td>
                                                </tr>
                                            )
                                        })}
                                        {/* SEPARATION VISUELLE MODIFIÉE (MOE/MOA en bas, Date en haut) */}
                                        <tr className="bg-gray-200">
                                            <td colSpan="8" className="border border-black font-bold text-center py-2">
                                                <div className="text-red-600 font-bold border-b border-gray-400 pb-1 mb-1 inline-block px-4">
                                                    PROCHAINE RÉUNION LE : {project.crInfo.nextMeetingDate ? new Date(project.crInfo.nextMeetingDate).toLocaleDateString('fr-FR') : '...'}
                                                </div>
                                                <div className="uppercase tracking-wider mt-1">MAÎTRISE D'OEUVRE / MAÎTRISE D'OUVRAGE</div>
                                            </td>
                                        </tr>
                                        {project.companies.filter(c => c.role !== 'ENT').map(c => {
                                            const showPresence = c.isConvocated !== false;
                                            return (
                                                <tr key={c.id} className="bg-gray-50">
                                                    <td className={`${cellClass} font-bold text-blue-900`}>{c.role}</td>
                                                    <td className={cellClass}><span className="font-bold">{c.name}</span><br/>{c.contact}</td>
                                                    <td className={`${cellClass} text-center font-bold`}>{showPresence && c.presence === 'P' ? 'X' : ''}</td>
                                                    <td className={`${cellClass} text-center font-bold text-red-600`}>{showPresence && c.presence === 'A' ? 'X' : ''}</td>
                                                    <td className={`${cellClass} text-center`}>{showPresence && c.presence === 'Ex' ? 'X' : ''}</td>
                                                    <td className={`${cellClass} text-center`}>{c.absences}</td>
                                                    <td className={`${cellClass} text-center font-bold text-red-600`}>{showPresence ? c.convocation : '-'}</td>
                                                    <td className={`${cellClass} whitespace-pre-line`}>{c.details}</td>
                                                </tr>
                                            )
                                        })}
                                    </tbody>
                                </table>
                            </div>
                        );
                    case 'general':
                        return (
                            <div className="mb-6">
                                <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-1" style={{ backgroundColor: primaryColor }}>Généralités & Sécurité</h2>
                                <div className="border border-gray-400 p-2 mb-4 text-[9px]">
                                    {project.generalities.map(g => (
                                        <div key={g.id} className="mb-2">
                                            <span className="font-bold underline">{g.title} :</span> {g.content}
                                        </div>
                                    ))}
                                    {project.delays.length > 0 && (
                                        <div className="mt-4 border-t border-gray-300 pt-2">
                                            <div className="font-bold text-red-600 uppercase mb-1">Intempéries / Retards</div>
                                            {project.delays.map(d => (
                                            <div key={d.id} className="flex justify-between">
                                                <span>{new Date(d.date).toLocaleDateString()} - {d.type}</span>
                                                <span className="italic">{d.reason}</span>
                                            </div>
                                            ))}
                                        </div>
                                    )}
                                </div>
                            </div>
                        );
                    case 'planning':
                        return (
                            <div className="mb-6">
                                <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-1" style={{ backgroundColor: primaryColor }}>Planning Prévisionnel</h2>
                                <table className={`w-full border-collapse border border-gray-400 ${density === 'compact' ? 'text-[8px]' : 'text-[9px]'}`}>
                                    <thead>
                                        <tr className="bg-gray-100">
                                            <th className="border p-1 text-left w-1/2">Tâche</th>
                                            <th className="border p-1 text-center w-16">Début</th>
                                            <th className="border p-1 text-center w-10">Durée</th>
                                            <th className="border p-1 text-center w-16">Fin</th>
                                            <th className="border p-1 text-center w-10">%</th>
                                        </tr>
                                    </thead>
                                    <tbody>
                                        {project.planning.sort((a,b) => new Date(a.dateStart) - new Date(b.dateStart)).map(p => (
                                            <tr key={p.id}>
                                            <td className="border p-1">{p.task} <span className="text-[8px] text-gray-500">({p.lotId ? p.lotId : 'Général'})</span></td>
                                            <td className="border p-1 text-center">{p.dateStart ? new Date(p.dateStart).toLocaleDateString() : '-'}</td>
                                            <td className="border p-1 text-center">{p.duration ? p.duration + 'j' : '-'}</td>
                                            <td className="border p-1 text-center">{p.dateEnd ? new Date(p.dateEnd).toLocaleDateString() : '-'}</td>
                                            <td className="border p-1 text-center font-bold">{p.progress}%</td>
                                            </tr>
                                        ))}
                                    </tbody>
                                </table>
                            </div>
                        );
                    case 'remarks':
                        return (
                            <div className="mb-6">
                                <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-2" style={{ backgroundColor: primaryColor }}>Remarques & Observations</h2>
                                {[
                                    { id: 'MOA', title: "MAÎTRISE D'OUVRAGE" },
                                    { id: 'MOE', title: "MAÎTRISE D'OEUVRE" },
                                    { id: 'EXT', title: "INTERVENANTS EXTÉRIEURS" }
                                ].map(section => {
                                    const rems = project.remarks.filter(r => r.type === section.id);
                                    if(rems.length === 0) return null;
                                    return (
                                        <div key={section.id} className="mb-4 avoid-break">
                                            <div className="font-bold border-b border-black mb-1" style={{ borderColor: primaryColor }}>{section.title}</div>
                                            <RemarkTable remarks={rems} />
                                        </div>
                                    )
                                })}
                                <div className="font-bold border-b border-black mb-2 mt-4" style={{ borderColor: primaryColor }}>ENTREPRISES</div>
                                {project.companies.filter(c => c.role === 'ENT').map(c => {
                                    const rems = project.remarks.filter(r => r.type === 'ENT' && r.lotId === c.id);
                                    if (rems.length === 0 && !showEmptyCompanies) return null;

                                    return (
                                        <div key={c.id} className="mb-4 avoid-break">
                                            <div className="bg-gray-200 px-2 py-0.5 font-bold flex justify-between">
                                            <span>{c.id} - {c.name} ({c.lot})</span>
                                            </div>
                                            {rems.length > 0 ? <RemarkTable remarks={rems} /> : <div className="text-[9px] italic p-1 border border-t-0 border-gray-300">Sans observation</div>}
                                        </div>
                                    )
                                })}
                            </div>
                        );
                    case 'choices':
                        return (
                            <div className="mb-6 avoid-break">
                                <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-1 w-full block" style={{ backgroundColor: primaryColor }}>Choix Validés</h2>
                                <ul className="list-disc pl-4 text-[9px]">
                                    {project.choices.map(c => <li key={c.id}><strong>{c.lotId ? `${c.lotId} : ` : ''}</strong>{c.description}</li>)}
                                </ul>
                            </div>
                        );
                    case 'photos':
                        return (
                            <div className="mb-6 avoid-break">
                                <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-1" style={{ backgroundColor: primaryColor }}>Photos</h2>
                                <div className="grid grid-cols-3 gap-1">
                                    {project.photos.slice(0, 6).map(p => (
                                        <div key={p.id}>
                                            <img src={p.url} className="w-full h-32 object-cover border" />
                                            <div className="text-[8px] mt-1">{p.caption}</div>
                                        </div>
                                    ))}
                                </div>
                            </div>
                        );
                    case 'footer':
                        return (
                            <div className="mt-8 pt-4 border-t border-gray-900 text-[9px] text-center text-gray-500">
                                <p>P: Présent - A: Absent - D: Diffusion - C: Convoqué | Rédaction : {footerAuthor}</p>
                                <p>{footerLegal}</p>
                            </div>
                        );
                    default:
                        const customSection = project.customSections.find(s => s.id === sectionId);
                        if (customSection) {
                            return (
                                <div className="mb-6 avoid-break">
                                    <h2 className="text-white px-2 py-1 font-bold uppercase text-xs mb-1" style={{ backgroundColor: primaryColor }}>{customSection.title}</h2>
                                    <div className="border border-gray-400 p-2 text-[9px] whitespace-pre-wrap">
                                        {customSection.content}
                                    </div>
                                </div>
                            );
                        }
                        return null;
                }
            };

            return (
            <div className={`bg-gray-500 p-8 min-h-screen flex justify-center overflow-auto print:p-0 print:bg-white print:h-auto font-sans leading-tight ${density === 'compact' ? 'text-[9px]' : 'text-[10px]'}`}>
                <div className="bg-white shadow-2xl w-[210mm] min-h-[297mm] p-[10mm] print:shadow-none print:w-full print:p-0">
                    {sections.filter(s => s.visible).map(s => (
                        <div key={s.id}>{renderSection(s.id)}</div>
                    ))}
                </div>
            </div>
            );
        };
        
        // --- APP PRINCIPALE ---
        const App = () => {
            const [projects, setProjects] = useState([]);
            const [currentProjectId, setCurrentProjectId] = useState(null);
            const [activeTab, setActiveTab] = useState('dashboard');
            const [showNewProjectModal, setShowNewProjectModal] = useState(false);
            const [isLoading, setIsLoading] = useState(true);
            const [showPdfHelp, setShowPdfHelp] = useState(false);
            
            // ... (Refs identiques)
            const fileInputRef = useRef(null);
            const photoInputRef = useRef(null);
            
            const currentProject = projects.find(p => p.id === currentProjectId) || null;

            // Initial Loading
            useEffect(() => {
                const init = async () => {
                    try {
                        const loadedProjects = await dbHelper.getAll();
                        if (loadedProjects.length === 0) {
                            await dbHelper.save(defaultProject);
                            setProjects([defaultProject]);
                        } else {
                            setProjects(loadedProjects);
                        }
                    } catch (err) {
                        console.error("DB Load Error:", err);
                    } finally {
                        setIsLoading(false);
                    }
                };
                init();
            }, []);

            // ... (Fonctions updateProject, createProject, etc. identiques au bloc React précédent)
            const updateProject = async (updates) => {
                if (!currentProject) return;
                const updatedProject = { ...currentProject, ...updates };
                setProjects(prev => prev.map(p => p.id === currentProjectId ? updatedProject : p));
                try { await dbHelper.save(updatedProject); } catch (err) { console.error(err); }
            };

            const createProject = async (name) => {
                if (name) {
                    const newProj = { ...defaultProject, id: Date.now().toString(), name, crInfo: { ...defaultProject.crInfo, number: 1, date: new Date().toISOString().split('T')[0] } };
                    await dbHelper.save(newProj);
                    setProjects(prev => [...prev, newProj]);
                    setShowNewProjectModal(false);
                }
            };
            
            const deleteProject = async (e, id) => {
                e.stopPropagation();
                if(confirm("Supprimer définitivement ce chantier ?")) {
                    await dbHelper.delete(id);
                    setProjects(prev => prev.filter(p => p.id !== id));
                    if (currentProjectId === id) setCurrentProjectId(null);
                }
            };

            const handleLogoUpload = (e) => {
                const file = e.target.files[0];
                if (file) resizeImage(file, 200).then(base64 => updateProject({ logo: base64 }));
            };

            const handlePhotoUpload = (e) => {
                const file = e.target.files[0];
                if (file) resizeImage(file, 800).then(base64 => {
                    updateProject({ photos: [...currentProject.photos, { id: Date.now(), url: base64, caption: '' }] });
                });
            };
            
            const addRemarkParent = (newRemarkData) => {
                if (!newRemarkData.text) return;
                const crNum = currentProject.crInfo.number;
                const countForCR = currentProject.remarks.filter(r => r.number.startsWith(`${crNum}.`)).length;
                const nextNum = `${crNum}.${countForCR + 1}`;
                updateProject({ remarks: [...currentProject.remarks, { id: Date.now(), ...newRemarkData, number: nextNum, status: 'normal', dateDone: '' }] });
            };

            const togglePresence = (compId, status) => {
                const updatedCompanies = currentProject.companies.map(c => {
                    if (c.id === compId) {
                        let newAbsences = c.absences;
                        if (status === 'A' && c.presence !== 'A') newAbsences += 1;
                        return { ...c, presence: status, absences: newAbsences };
                    }
                    return c;
                });
                updateProject({ companies: updatedCompanies });
            };
            
            // MODALE HELP PDF
            const PdfHelpModal = () => (
                <div className="fixed inset-0 bg-black/50 z-50 flex items-center justify-center p-4">
                    <div className="bg-white rounded-xl shadow-2xl p-6 max-w-lg w-full text-left">
                        <div className="flex justify-between items-center mb-4">
                            <h3 className="text-xl font-bold flex items-center gap-2 text-gray-900"><FileText className="text-red-500"/> Export PDF</h3>
                            <button onClick={() => setShowPdfHelp(false)} className="text-gray-400 hover:text-black"><X size={24}/></button>
                        </div>
                        <p className="mb-4 text-gray-600">
                            Pour enregistrer votre rapport en PDF :
                        </p>
                        <ol className="list-decimal pl-5 space-y-3 mb-6 text-sm text-gray-800">
                            <li>Cliquez sur le bouton <strong>"Imprimer maintenant"</strong> ci-dessous.</li>
                            <li>Dans la fenêtre d'impression, changez la <strong>Destination</strong> pour choisir <strong>"Enregistrer au format PDF"</strong>.</li>
                            <li>Dans "Plus de paramètres", cochez <strong>"Graphiques d'arrière-plan"</strong> (pour les couleurs).</li>
                            <li>Cliquez sur <strong>Enregistrer</strong>.</li>
                        </ol>
                        <div className="flex justify-end gap-2">
                            <Button variant="secondary" onClick={() => setShowPdfHelp(false)}>Annuler</Button>
                            <Button onClick={() => { setShowPdfHelp(false); setTimeout(() => window.print(), 100); }} icon={Printer}>Imprimer maintenant</Button>
                        </div>
                    </div>
                </div>
            );

            const NewProjectModal = () => {
                const [name, setName] = useState('');
                return (
                    <div className="fixed inset-0 bg-black/50 z-50 flex items-center justify-center p-4">
                        <div className="bg-white rounded-xl shadow-2xl p-6 max-w-sm w-full">
                            <h3 className="text-lg font-bold mb-4">Nouveau Chantier</h3>
                            <Input label="Nom" value={name} onChange={setName} placeholder="Nom du chantier..." />
                            <div className="flex justify-end gap-2 mt-4">
                                <Button variant="secondary" onClick={() => setShowNewProjectModal(false)}>Annuler</Button>
                                <Button onClick={() => createProject(name)}>Créer</Button>
                            </div>
                        </div>
                    </div>
                );
            }

            if (isLoading) return <div className="h-screen flex items-center justify-center bg-gray-50"><Loader2 className="animate-spin text-blue-600" size={40} /></div>;

            if (!currentProject) return (
                <div className="p-8 max-w-4xl mx-auto h-screen overflow-y-auto">
                    <h1 className="text-3xl font-bold text-gray-900 mb-6">Mes Chantiers</h1>
                    {showNewProjectModal && <NewProjectModal />}
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                        {projects.map(p => (
                        <div key={p.id} onClick={() => { setCurrentProjectId(p.id); setActiveTab('config'); }} className="bg-white p-6 rounded-xl shadow cursor-pointer hover:border-blue-500 border-2 border-transparent transition-all group relative">
                            <h3 className="font-bold text-xl mb-1">{p.name}</h3>
                            <p className="text-gray-500 text-sm">Dernier CR : N°{p.crInfo.number}</p>
                            <div className="mt-4 flex gap-2">
                                <span className="text-xs bg-gray-100 px-2 py-1 rounded text-gray-600">{p.companies.length} Ent.</span>
                                <span className="text-xs bg-gray-100 px-2 py-1 rounded text-gray-600">{p.remarks.length} Rem.</span>
                            </div>
                            <button onClick={(e) => deleteProject(e, p.id)} className="absolute top-4 right-4 text-gray-300 hover:text-red-500"><Trash2 size={18} /></button>
                        </div>
                        ))}
                        <button onClick={() => setShowNewProjectModal(true)} className="border-2 border-dashed border-gray-300 rounded-xl p-6 flex flex-col items-center justify-center text-gray-400 hover:text-blue-500 hover:border-blue-500 transition-all bg-gray-50 min-h-[150px]">
                        <Plus size={40} />
                        <span className="font-bold mt-2">Nouveau Chantier</span>
                        </button>
                    </div>
                </div>
            );

            return (
                <div className="flex h-screen bg-gray-100 font-sans text-gray-900 overflow-hidden">
                    {showPdfHelp && <PdfHelpModal />}
                    
                    <div className="w-64 bg-gray-900 text-white flex flex-col shadow-xl print-hidden z-10">
                        <div className="p-4 border-b border-gray-800 cursor-pointer hover:bg-gray-800" onClick={() => setCurrentProjectId(null)}>
                        <div className="flex items-center gap-2 text-gray-400 text-xs mb-1"><ArrowLeft size={12} /> CHANTIERS</div>
                        <h1 className="font-bold leading-tight truncate">{currentProject.name}</h1>
                        </div>
                        <nav className="flex-1 p-2 space-y-1 overflow-y-auto">
                        <button onClick={() => setActiveTab('config')} className={`w-full flex items-center gap-3 px-3 py-2 rounded-lg text-sm transition-all mb-2 ${activeTab === 'config' ? 'bg-orange-500 text-white' : 'bg-gray-800 text-gray-300 hover:text-white'}`}>
                            <Settings size={18} /> Configuration
                        </button>
                        <div className="border-t border-gray-700 my-2"></div>
                        {[
                            { id: 'dashboard', icon: LayoutDashboard, label: 'Tableau de Bord' },
                            { id: 'attendance', icon: Users, label: 'Présences' },
                            { id: 'general', icon: AlertTriangle, label: 'Généralités' },
                            { id: 'planning', icon: Calendar, label: 'Planning' },
                            { id: 'remarks', icon: MessageSquare, label: 'Remarques' },
                            { id: 'choices', icon: CheckSquare, label: 'Choix' },
                            { id: 'photos', icon: Camera, label: 'Photos' },
                        ].map(item => (
                            <button key={item.id} onClick={() => setActiveTab(item.id)} className={`w-full flex items-center gap-3 px-3 py-2 rounded-lg text-sm transition-all ${activeTab === item.id ? 'bg-blue-600 text-white' : 'hover:bg-gray-800 text-gray-400'}`}>
                                <item.icon size={18} /> {item.label}
                            </button>
                        ))}
                         <button onClick={() => setActiveTab('print')} className={`w-full flex items-center gap-3 px-3 py-2 rounded-lg text-sm transition-all mt-4 ${activeTab === 'print' ? 'bg-blue-600 text-white' : 'bg-blue-800 hover:bg-blue-700 text-white'}`}>
                            <FileText size={18} /> Export PDF
                        </button>
                        </nav>
                    </div>

                    <div className="flex-1 overflow-auto relative bg-gray-50" id="main-content">
                        {activeTab === 'print' ? (
                        <>
                            <div className="fixed bottom-8 right-8 print-hidden z-50 flex flex-col gap-2">
                                <button onClick={() => setShowPdfHelp(true)} className="bg-white text-blue-600 p-3 rounded-full shadow-lg hover:bg-blue-50 transition-all" title="Aide PDF">
                                    <HelpCircle size={24}/>
                                </button>
                                <Button onClick={() => setShowPdfHelp(true)} variant="primary" icon={Printer} className="shadow-2xl scale-125 origin-bottom-right px-6 py-3 text-lg">Imprimer / PDF</Button>
                            </div>
                            <PrintPreview project={currentProject} />
                        </>
                        ) : (
                        <div className="p-8 max-w-6xl mx-auto pb-20">
                            <header className="flex justify-between items-center mb-6">
                                <h2 className="text-2xl font-bold text-gray-800 capitalize">{activeTab === 'config' ? 'Configuration & Personnalisation' : activeTab}</h2>
                            </header>
                            <div className="animate-in fade-in slide-in-from-bottom-2 duration-300">
                            {activeTab === 'dashboard' && (
                                <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
                                    <Card title="État" className="md:col-span-2">
                                    <div className="text-4xl font-black text-blue-600">CR N°{currentProject.crInfo.number}</div>
                                    <div className="text-gray-500">Réunion du {new Date(currentProject.crInfo.date).toLocaleDateString()}</div>
                                    </Card>
                                    <Card title="Statistiques">
                                    <div className="flex justify-between border-b py-2"><span>Entreprises</span> <strong>{currentProject.companies.length}</strong></div>
                                    <div className="flex justify-between py-2"><span>Remarques</span> <strong>{currentProject.remarks.length}</strong></div>
                                    </Card>
                                </div>
                            )}
                            {activeTab === 'config' && <ConfigView project={currentProject} updateProject={updateProject} fileInputRef={fileInputRef} handleLogoUpload={handleLogoUpload} />}
                            {activeTab === 'attendance' && <AttendanceView project={currentProject} togglePresence={togglePresence} updateProject={updateProject} />}
                            {activeTab === 'general' && <GeneralitiesView project={currentProject} updateProject={updateProject} />}
                            {activeTab === 'planning' && <PlanningView project={currentProject} updateProject={updateProject} />}
                            {activeTab === 'remarks' && <RemarksView project={currentProject} updateProject={updateProject} addRemarkParent={addRemarkParent} />}
                            {activeTab === 'choices' && <ChoicesView project={currentProject} updateProject={updateProject} />}
                            {activeTab === 'photos' && <PhotosView project={currentProject} updateProject={updateProject} handlePhotoUpload={handlePhotoUpload} photoInputRef={photoInputRef} />}
                            </div>
                        </div>
                        )}
                    </div>
                </div>
            );
        };

        const root = ReactDOM.createRoot(document.getElementById('root'));
        root.render(<App />);
    </script>
</body>
</html>
