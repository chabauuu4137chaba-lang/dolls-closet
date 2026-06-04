import React, { useState, useRef, useEffect } from 'react';
import { 
  Heart, 
  Trash2, 
  Download, 
  Upload, 
  Move, 
  Layers, 
  Sparkles, 
  RotateCw, 
  Grid, 
  ZoomIn, 
  ZoomOut, 
  Undo,
  Check,
  Plus,
  Sliders,
  Scissors,
  HelpCircle,
  Flower,
  FolderHeart,
  Tag
} from 'lucide-react';

export default function App() {
  // --- アプリ全体のステート ---
  const [images, setImages] = useState([]); // キャンバス上のコーディネート
  const [selectedImageId, setSelectedImageId] = useState(null); // 選択中のお洋服ID
  const [isEraserMode, setIsEraserMode] = useState(false); // 消しゴム起動フラグ
  const [eraserSize, setEraserSize] = useState(25); // ブラシサイズ
  const [eraserHardness, setEraserHardness] = useState(0.3); // ぼかし加減
  const [canvasZoom, setCanvasZoom] = useState(1); // ズーム倍率

  // --- 【新機能】仕立て済みお洋服コレクション（カテゴリー対応） ---
  const [clothingCollection, setClothingCollection] = useState([]);
  const [leftPanelTab, setLeftPanelTab] = useState('editor'); // 'editor' or 'collection'
  const [selectedCategoryFilter, setSelectedCategoryFilter] = useState('all'); // 絞り込み用

  // --- 新しくお洋服を仕立てる時のカテゴリー選択用 ---
  const [newImageCategory, setNewImageCategory] = useState('dress'); // デフォルトはワンピース

  // --- 背景消しゴム用のステート ---
  const [editorImage, setEditorImage] = useState(null); 
  const [history, setHistory] = useState([]); 
  const [historyIndex, setHistoryIndex] = useState(-1);
  const [tolerance, setTolerance] = useState(35); 

  // --- 案内ダイアログ ---
  const [showGuide, setShowGuide] = useState(true);

  // --- Refs ---
  const fileInputRef = useRef(null);
  const eraserCanvasRef = useRef(null);
  const coordinateFieldRef = useRef(null);

  // --- カテゴリー定義 ---
  const categories = [
    { key: 'tops', label: 'トップス', icon: '👚' },
    { key: 'bottoms', label: 'ボトムス', icon: '👖' },
    { key: 'dress', label: 'ワンピース', icon: '👗' },
    { key: 'accessory', label: 'アクセサリー', icon: '🎀' }
  ];

  // --- 初期デモデータ ＆ コレクション初期ストック ---
  useEffect(() => {
    const demoDressUrl = 'https://images.unsplash.com/photo-1518600333142-4d3e334d450b?w=300&auto=format&fit=crop&q=60';
    const demoHeadUrl = 'https://images.unsplash.com/photo-1560845946-f2884d952a25?w=200&auto=format&fit=crop&q=60';
    const demoSkirtUrl = 'https://images.unsplash.com/photo-1583496661160-fb5886a0aaaa?w=300&auto=format&fit=crop&q=60'; // ピンクボトムス
    const demoBlouseUrl = 'https://images.unsplash.com/photo-1548624149-f9b39d734338?w=300&auto=format&fit=crop&q=60'; // ホワイトトップス

    // キャンバス上の初期配置
    const demoItems = [
      {
        id: 'demo-dress',
        src: demoDressUrl,
        x: 120,
        y: 120,
        width: 220,
        height: 290,
        rotation: -2,
        zIndex: 2,
        name: 'クラシカルローズ・ドレス',
        category: 'dress'
      },
      {
        id: 'demo-head',
        src: demoHeadUrl,
        x: 180,
        y: 20,
        width: 110,
        height: 110,
        rotation: 8,
        zIndex: 3,
        name: 'ヴィクトリアン・ボンネット',
        category: 'accessory'
      }
    ];
    setImages(demoItems);

    // コレクションの初期分類ストック
    setClothingCollection([
      {
        id: 'stock-dress-1',
        src: demoDressUrl,
        name: 'クラシカルローズ・ドレス',
        category: 'dress',
        date: '2026/06/04'
      },
      {
        id: 'stock-head-1',
        src: demoHeadUrl,
        name: 'ヴィクトリアン・ボンネット',
        category: 'accessory',
        date: '2026/06/04'
      },
      {
        id: 'stock-tops-1',
        src: demoBlouseUrl,
        name: 'フリルシャーリング・ブラウス',
        category: 'tops',
        date: '2026/06/04'
      },
      {
        id: 'stock-bottoms-1',
        src: demoSkirtUrl,
        name: 'ティアード・パニエスカート',
        category: 'bottoms',
        date: '2026/06/04'
      }
    ]);
  }, []);

  // --- 画像アップロード ---
  const handleImageUpload = (e) => {
    const file = e.target.files[0];
    if (file) {
      const reader = new FileReader();
      reader.onload = (event) => {
        setEditorImage(event.target.result);
        setIsEraserMode(true);
        setLeftPanelTab('editor'); // エディタタブへ
      };
      reader.readAsDataURL(file);
    }
  };

  // --- 消しゴムキャンバス初期化 ---
  useEffect(() => {
    if (editorImage && eraserCanvasRef.current) {
      const canvas = eraserCanvasRef.current;
      const ctx = canvas.getContext('2d');
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload = () => {
        const maxDim = 520;
        let w = img.width;
        let h = img.height;
        if (w > h) {
          if (w > maxDim) {
            h = (h * maxDim) / w;
            w = maxDim;
          }
        } else {
          if (h > maxDim) {
            w = (w * maxDim) / h;
            h = maxDim;
          }
        }
        canvas.width = w;
        canvas.height = h;
        
        ctx.clearRect(0, 0, w, h);
        ctx.drawImage(img, 0, 0, w, h);

        const initialState = ctx.getImageData(0, 0, w, h);
        setHistory([initialState]);
        setHistoryIndex(0);
      };
      img.src = editorImage;
    }
  }, [editorImage]);

  // --- スマホ画面固定化スクロール防止 ---
  useEffect(() => {
    const canvas = eraserCanvasRef.current;
    if (!canvas) return;

    const preventScroll = (e) => {
      if (isDrawingRef.current) {
        e.preventDefault();
      }
    };

    const preventWheel = (e) => {
      if (isDrawingRef.current) {
        e.preventDefault();
      }
    };

    canvas.addEventListener('touchmove', preventScroll, { passive: false });
    canvas.addEventListener('touchstart', preventScroll, { passive: false });
    canvas.addEventListener('wheel', preventWheel, { passive: false });

    return () => {
      canvas.removeEventListener('touchmove', preventScroll);
      canvas.removeEventListener('touchstart', preventScroll);
      canvas.removeEventListener('wheel', preventWheel);
    };
  }, [editorImage]);

  // --- 消しゴム描画ロジック ---
  const isDrawingRef = useRef(false);

  const getCanvasCoords = (e) => {
    const canvas = eraserCanvasRef.current;
    if (!canvas) return { x: 0, y: 0 };
    const rect = canvas.getBoundingClientRect();
    
    let clientX = 0;
    let clientY = 0;

    if (e.touches && e.touches.length > 0) {
      clientX = e.touches[0].clientX;
      clientY = e.touches[0].clientY;
    } else if (e.changedTouches && e.changedTouches.length > 0) {
      clientX = e.changedTouches[0].clientX;
      clientY = e.changedTouches[0].clientY;
    } else {
      clientX = e.clientX;
      clientY = e.clientY;
    }
    
    return {
      x: (clientX - rect.left) * (canvas.width / rect.width),
      y: (clientY - rect.top) * (canvas.height / rect.height)
    };
  };

  const handleStartErase = (e) => {
    isDrawingRef.current = true;
    erase(e);
  };

  const handleEraseMove = (e) => {
    if (!isDrawingRef.current) return;
    erase(e);
  };

  const handleStopErase = () => {
    if (isDrawingRef.current) {
      isDrawingRef.current = false;
      const canvas = eraserCanvasRef.current;
      if (canvas) {
        const ctx = canvas.getContext('2d');
        const currentState = ctx.getImageData(0, 0, canvas.width, canvas.height);
        const newHistory = history.slice(0, historyIndex + 1);
        setHistory([...newHistory, currentState]);
        setHistoryIndex(newHistory.length);
      }
    }
  };

  const erase = (e) => {
    const canvas = eraserCanvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const coords = getCanvasCoords(e);

    ctx.save();
    const gradient = ctx.createRadialGradient(
      coords.x, coords.y, eraserSize * eraserHardness,
      coords.x, coords.y, eraserSize
    );
    
    gradient.addColorStop(0, 'rgba(0, 0, 0, 1.0)');
    gradient.addColorStop(0.5, 'rgba(0, 0, 0, 0.5)');
    gradient.addColorStop(1, 'rgba(0, 0, 0, 0.0)');

    ctx.globalCompositeOperation = 'destination-out';
    ctx.fillStyle = gradient;
    ctx.beginPath();
    ctx.arc(coords.x, coords.y, eraserSize, 0, Math.PI * 2, false);
    ctx.fill();
    ctx.restore();
  };

  // --- 高精度自動透過 ---
  const handleAutoColorErase = () => {
    const canvas = eraserCanvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imgData.data;

    const samplePixels = [0, 4, canvas.width * 4, canvas.width * 4 + 4];
    let sumR = 0, sumG = 0, sumB = 0;
    samplePixels.forEach(idx => {
      sumR += data[idx];
      sumG += data[idx + 1];
      sumB += data[idx + 2];
    });
    const targetR = sumR / samplePixels.length;
    const targetG = sumG / samplePixels.length;
    const targetB = sumB / samplePixels.length;

    const featherWidth = 15; 

    for (let i = 0; i < data.length; i += 4) {
      const r = data[i];
      const g = data[i + 1];
      const b = data[i + 2];
      const a = data[i + 3];

      if (a === 0) continue;

      const diff = Math.sqrt(
        Math.pow(r - targetR, 2) +
        Math.pow(g - targetG, 2) +
        Math.pow(b - targetB, 2)
      );

      if (diff < tolerance) {
        data[i + 3] = 0; 
      } 
      else if (diff < tolerance + featherWidth) {
        const ratio = (diff - tolerance) / featherWidth;
        data[i + 3] = Math.min(data[i + 3], ratio * 255);
      }
    }

    ctx.putImageData(imgData, 0, 0);
    
    const newHistory = history.slice(0, historyIndex + 1);
    setHistory([...newHistory, imgData]);
    setHistoryIndex(newHistory.length);
  };

  // --- Undo ---
  const handleUndo = () => {
    if (historyIndex > 0) {
      const newIndex = historyIndex - 1;
      setHistoryIndex(newIndex);
      const canvas = eraserCanvasRef.current;
      if (canvas) {
        const ctx = canvas.getContext('2d');
        ctx.putImageData(history[newIndex], 0, 0);
      }
    }
  };

  // --- 仕立てたお洋服をコレクションに登録 (指定のカテゴリーに) ---
  const handleAddToCanvas = () => {
    const canvas = eraserCanvasRef.current;
    if (!canvas) return;
    
    const erasedDataUrl = canvas.toDataURL('image/png');
    const itemId = `doll-item-${Date.now()}`;
    const matchedCat = categories.find(c => c.key === newImageCategory);
    const catLabel = matchedCat ? matchedCat.label : 'お洋服';
    const defaultName = `仕立てた${catLabel} #${clothingCollection.filter(i => i.category === newImageCategory).length + 1}`;

    // 1. カテゴリー情報を付与してコレクションにストック
    const newStockItem = {
      id: itemId,
      src: erasedDataUrl,
      name: defaultName,
      category: newImageCategory,
      date: new Date().toLocaleDateString()
    };
    setClothingCollection([newStockItem, ...clothingCollection]);

    // 2. 現在のコーディネート画面へ自動配置
    const newCanvasItem = {
      id: itemId,
      src: erasedDataUrl,
      x: 80,
      y: 100,
      width: 220,
      height: 220,
      rotation: 0,
      zIndex: images.length + 1,
      name: defaultName,
      category: newImageCategory
    };
    setImages([...images, newCanvasItem]);
    setSelectedImageId(itemId);
    
    // エディタをクリアし、仕立てた一覧の同じカテゴリーのタブをアクティブにする
    setEditorImage(null);
    setIsEraserMode(false);
    setSelectedCategoryFilter(newImageCategory);
    setLeftPanelTab('collection');
  };

  // --- コレクションからコーディネートに「着せる」 ---
  const handlePutOnFromCollection = (item) => {
    const uniqueId = `cloned-item-${Date.now()}`;
    const newItem = {
      id: uniqueId,
      src: item.src,
      x: 100 + (images.length * 20) % 120, 
      y: 120 + (images.length * 20) % 120,
      width: 200,
      height: 200,
      rotation: 0,
      zIndex: images.length + 1,
      name: item.name,
      category: item.category
    };
    setImages([...images, newItem]);
    setSelectedImageId(uniqueId);
  };

  // --- コレクションから削除 ---
  const handleDeleteFromCollection = (id, e) => {
    e.stopPropagation();
    if (window.confirm('このお洋服をクローゼットのコレクションから完全に削除しますか？')) {
      setClothingCollection(clothingCollection.filter(item => item.id !== id));
    }
  };

  // --- お洋服の名前変更 ---
  const handleRenameCollectionItem = (id, newName) => {
    setClothingCollection(clothingCollection.map(item => {
      if (item.id === id) {
        return { ...item, name: newName || 'なまえなしのアイテム' };
      }
      return item;
    }));
  };

  // --- コーディネートフィールドのドラッグ処理 ---
  const [dragStart, setDragStart] = useState(null);
  const [isDragging, setIsDragging] = useState(false);

  const handleCanvasItemMouseDown = (id, e) => {
    e.stopPropagation();
    setSelectedImageId(id);
    setIsDragging(true);
    
    const clientX = e.touches ? e.touches[0].clientX : e.clientX;
    const clientY = e.touches ? e.touches[0].clientY : e.clientY;
    
    const targetItem = images.find(img => img.id === id);
    setDragStart({
      startX: clientX,
      startY: clientY,
      itemX: targetItem.x,
      itemY: targetItem.y,
    });
  };

  const handleCanvasMouseMove = (e) => {
    if (isDragging && dragStart && selectedImageId) {
      const clientX = e.touches ? e.touches[0].clientX : e.clientX;
      const clientY = e.touches ? e.touches[0].clientY : e.clientY;
      
      const dx = (clientX - dragStart.startX) / canvasZoom;
      const dy = (clientY - dragStart.startY) / canvasZoom;

      setImages(images.map(img => {
        if (img.id === selectedImageId) {
          return {
            ...img,
            x: dragStart.itemX + dx,
            y: dragStart.itemY + dy,
          };
        }
        return img;
      }));
    }
  };

  const handleCanvasMouseUp = () => {
    setIsDragging(false);
  };

  // --- フィールド内の各種調整 ---
  const handleTransformSelected = (action, value) => {
    if (!selectedImageId) return;
    setImages(images.map(img => {
      if (img.id === selectedImageId) {
        if (action === 'rotate') {
          return { ...img, rotation: (img.rotation + value) % 360 };
        }
        if (action === 'resize') {
          const newWidth = Math.max(40, img.width + value);
          const newHeight = Math.max(40, img.height + value);
          return { ...img, width: newWidth, height: newHeight };
        }
        if (action === 'layer') {
          return { ...img, zIndex: Math.max(1, img.zIndex + value) };
        }
      }
      return img;
    }));
  };

  const handleDeleteSelected = () => {
    if (!selectedImageId) return;
    setImages(images.filter(img => img.id !== selectedImageId));
    setSelectedImageId(null);
  };

  const handleClearCanvas = () => {
    if(window.confirm('コーディネートキャンバスに飾っているお洋服をすべて片付けますか？ (コレクションは消えません)')) {
      setImages([]);
      setSelectedImageId(null);
    }
  };

  // --- クローゼット保存 ---
  const handleDownloadCanvas = () => {
    const field = coordinateFieldRef.current;
    if (!field) return;

    const canvas = document.createElement('canvas');
    canvas.width = 900;
    canvas.height = 1000;
    const ctx = canvas.getContext('2d');
    
    ctx.fillStyle = '#fff9fa';
    ctx.fillRect(0, 0, canvas.width, canvas.height);

    ctx.strokeStyle = '#e6cfb3';
    ctx.lineWidth = 14;
    ctx.strokeRect(15, 15, canvas.width - 30, canvas.height - 30);
    ctx.strokeStyle = '#d4af37';
    ctx.lineWidth = 2;
    ctx.strokeRect(25, 25, canvas.width - 50, canvas.height - 50);

    const sortedImages = [...images].sort((a, b) => a.zIndex - b.zIndex);
    if (sortedImages.length === 0) {
      alert("キャンバスにお洋服が配置されていません。");
      return;
    }

    let loadedCount = 0;
    sortedImages.forEach(imgData => {
      const img = new Image();
      img.crossOrigin = 'anonymous';
      img.onload = () => {
        ctx.save();
        const cx = imgData.x + imgData.width / 2;
        const cy = imgData.y + imgData.height / 2;
        ctx.translate(cx, cy);
        ctx.rotate((imgData.rotation * Math.PI) / 180);
        ctx.drawImage(
          img, 
          -imgData.width / 2, 
          -imgData.height / 2, 
          imgData.width, 
          imgData.height
        );
        ctx.restore();

        loadedCount++;
        if (loadedCount === sortedImages.length) {
          const link = document.createElement('a');
          link.download = 'maison-de-lolita-coordinate.png';
          link.href = canvas.toDataURL('image/png');
          link.click();
        }
      };
      img.src = imgData.src;
    });
  };

  // コレクション絞り込みフィルター
  const filteredCollection = selectedCategoryFilter === 'all'
    ? clothingCollection
    : clothingCollection.filter(item => item.category === selectedCategoryFilter);

  return (
    <div className="min-h-screen bg-[#faf3f3] text-stone-800 font-serif flex flex-col relative overflow-x-hidden">
      
      {/* チュチュレース風デコレーションリボン */}
      <div className="absolute top-0 left-0 right-0 h-2.5 bg-gradient-to-r from-pink-300 via-rose-200 to-pink-300 opacity-90 z-50"></div>
      
      {/* サロンの案内ポップアップ */}
      {showGuide && (
        <div className="fixed inset-0 bg-stone-900/40 backdrop-blur-sm flex items-center justify-center p-4 z-50 animate-fadeIn">
          <div className="bg-white rounded-3xl border-4 border-[#e9cbd1] max-w-md w-full p-6 shadow-2xl relative">
            <button 
              onClick={() => setShowGuide(false)}
              className="absolute top-3 right-3 text-stone-400 hover:text-pink-400 font-bold text-xl"
            >
              ×
            </button>
            <div className="text-center">
              <Heart className="w-8 h-8 text-pink-400 mx-auto fill-pink-100 mb-2" />
              <h4 className="font-bold text-lg text-stone-800 mb-1">Maison de l'Ange Wardrobe</h4>
              <p className="text-xs text-rose-400 italic mb-4">〜 クローゼットの整理整頓が整いました 〜</p>
            </div>
            <div className="space-y-2.5 text-xs text-stone-600 leading-relaxed font-sans">
              <p className="bg-[#fff6f7] p-2.5 rounded-xl border border-pink-100">
                <strong>👚 トップス・👖 ボトムス・👗 ドレス・🎀 アクセサリー</strong><br/>
                新しく透過したお洋服をお好みのカテゴリーを選んで保存できるようになりました。カテゴリー別タブですっきり一覧化されます！
              </p>
              <p className="bg-[#fff6f7] p-2.5 rounded-xl border border-pink-100">
                <strong>✂ スムーズな消しゴム</strong><br/>
                消しゴムなぞり中の画面スクロールは完全に制限され、お洋服仕立てに集中できます。
              </p>
            </div>
            <button
              onClick={() => setShowGuide(false)}
              className="mt-5 w-full py-2.5 bg-gradient-to-r from-pink-400 to-rose-300 text-white rounded-full font-bold shadow-md shadow-pink-200 hover:opacity-95 transition text-xs"
            >
              クローゼットを開く
            </button>
          </div>
        </div>
      )}

      {/* ヘッダー */}
      <header className="border-b border-pink-100 bg-[#fffbfc] px-6 py-5 flex flex-wrap justify-between items-center gap-4 relative shadow-sm">
        <div className="flex items-center gap-4">
          <div className="relative p-3 bg-gradient-to-b from-rose-100 to-pink-50 rounded-full border-2 border-pink-200/60 shadow-inner">
            <Heart className="w-6 h-6 text-pink-400 fill-pink-300 stroke-[1.5]" />
          </div>
          <div>
            <h1 className="text-2xl font-black tracking-widest text-[#6b4c4c] flex items-center gap-2">
              Maison de Poupée
            </h1>
            <p className="text-[11px] tracking-wider text-rose-400/90 italic font-sans flex items-center gap-1">
              <Flower className="w-3.5 h-3.5 text-pink-300" />
              クラシカル背景消しゴム ＆ 分類クローゼット
            </p>
          </div>
        </div>

        <div className="flex items-center gap-3">
          <button 
            onClick={() => fileInputRef.current?.click()}
            className="flex items-center gap-2 px-5 py-3 bg-gradient-to-r from-pink-400 to-rose-400 hover:from-pink-500 hover:to-rose-500 text-white text-xs font-bold rounded-full transition-all shadow-md shadow-pink-200"
          >
            <Scissors className="w-4 h-4 text-white/90" />
            <span>お洋服を仕立てる (背景消しゴム)</span>
          </button>
          <input 
            type="file"
            ref={fileInputRef}
            onChange={handleImageUpload}
            accept="image/*"
            className="hidden"
          />

          <button
            onClick={handleDownloadCanvas}
            className="flex items-center gap-1.5 px-4 py-3 bg-white hover:bg-pink-50/50 text-stone-600 text-xs font-bold rounded-full transition-all border-2 border-pink-100 shadow-sm"
          >
            <Download className="w-4 h-4 text-pink-400" />
            <span>クローゼットを画像保存</span>
          </button>
        </div>
      </header>

      {/* メインレイアウト */}
      <main className="flex-1 max-w-7xl w-full mx-auto p-4 md:p-6 grid grid-cols-1 lg:grid-cols-12 gap-6">
        
        {/* 左側操作パネル (4カラム) */}
        <div className="lg:col-span-5 flex flex-col gap-5">
          
          <div className="bg-white rounded-3xl border-2 border-pink-100 p-5 flex flex-col h-[650px] justify-between shadow-md relative">
            <div className="absolute top-0 left-0 right-0 h-1 bg-[radial-gradient(circle,transparent_20%,#fecdd3_20%,#fecdd3_40%,transparent_40%)] bg-[length:10px_10px]"></div>
            
            {/* メインタブ切替 */}
            <div className="flex border-b border-pink-100/80 mb-3 pt-2">
              <button
                onClick={() => setLeftPanelTab('editor')}
                className={`flex-1 py-2 text-xs font-bold tracking-wider border-b-2 transition ${
                  leftPanelTab === 'editor' 
                    ? 'border-pink-400 text-pink-600 font-extrabold' 
                    : 'border-transparent text-stone-400 hover:text-stone-600'
                }`}
              >
                ✂ お洋服の透過加工
              </button>
              <button
                onClick={() => setLeftPanelTab('collection')}
                className={`flex-1 py-2 text-xs font-bold tracking-wider border-b-2 transition flex items-center justify-center gap-1 ${
                  leftPanelTab === 'collection' 
                    ? 'border-pink-400 text-pink-600 font-extrabold' 
                    : 'border-transparent text-stone-400 hover:text-stone-600'
                }`}
              >
                <FolderHeart className="w-3.5 h-3.5" />
                <span>仕立てた服リスト ({clothingCollection.length})</span>
              </button>
            </div>

            {/* 各タブのコンテンツ */}
            <div className="flex-1 flex flex-col justify-between overflow-hidden">
              {leftPanelTab === 'editor' ? (
                // === 消しゴムエディタ タブ ===
                <div className="flex-1 flex flex-col justify-between h-full">
                  <div>
                    <p className="text-[11px] text-stone-500 font-sans leading-relaxed mb-2">
                      消しゴム境界線ソフト加工により、レースやお花の細かい部分もなめらかに透過できます。
                    </p>
                  </div>

                  <div className="flex-1 bg-[#fffbfb] rounded-2xl border-2 border-dashed border-pink-150 relative flex items-center justify-center overflow-hidden min-h-[290px] my-2 shadow-inner">
                    {editorImage ? (
                      <div className="relative group flex items-center justify-center p-4 w-full h-full select-none">
                        <canvas
                          ref={eraserCanvasRef}
                          onMouseDown={handleStartErase}
                          onMouseMove={handleEraseMove}
                          onMouseUp={handleStopErase}
                          onMouseLeave={handleStopErase}
                          onTouchStart={handleStartErase}
                          onTouchMove={handleEraseMove}
                          onTouchEnd={handleStopErase}
                          className="max-w-full max-h-full rounded-lg shadow-md cursor-crosshair border border-pink-100 bg-[radial-gradient(#e5e7eb_1px,transparent_1px)] [background-size:16px_16px] bg-white"
                          style={{
                            touchAction: 'none',
                            userSelect: 'none',
                            WebkitUserSelect: 'none'
                          }}
                        />
                        
                        <div className="absolute bottom-3 left-3 right-3 flex justify-between items-center pointer-events-none">
                          <span className="bg-white/95 text-[10px] text-pink-500 font-sans px-2.5 py-1.5 rounded-full border border-pink-100 shadow-sm backdrop-blur font-bold">
                            ✍ なぞって消去
                          </span>
                          <button
                            onClick={handleUndo}
                            disabled={historyIndex <= 0}
                            className="pointer-events-auto p-2.5 bg-white hover:bg-pink-50/50 disabled:opacity-30 disabled:cursor-not-allowed rounded-full border border-pink-100 text-[#5c3e3e] shadow-sm transition"
                            title="元に戻す"
                          >
                            <Undo className="w-4 h-4 text-pink-400" />
                          </button>
                        </div>
                      </div>
                    ) : (
                      <div className="text-center p-6 flex flex-col items-center justify-center h-full font-sans">
                        <div className="w-14 h-14 rounded-full bg-pink-50 flex items-center justify-center mb-3 border border-pink-100">
                          <Upload className="w-5 h-5 text-pink-400" />
                        </div>
                        <p className="text-xs font-bold text-stone-700">お洋服写真の切り抜き</p>
                        <p className="text-[10px] text-stone-400 mt-1 mb-4 max-w-[220px] leading-relaxed">
                          フリルやカチューシャ、私服の画像を仕立ててクローゼットのカテゴリーへ納品しましょう。
                        </p>
                        <button
                          onClick={() => fileInputRef.current?.click()}
                          className="px-5 py-2 bg-pink-50 hover:bg-pink-100 text-pink-600 text-xs font-bold rounded-full transition border border-pink-200"
                        >
                          画像をえらぶ
                        </button>
                      </div>
                    )}
                  </div>

                  {editorImage && (
                    <div className="space-y-2.5 font-sans">
                      
                      {/* 消しゴム基本パラメータ */}
                      <div className="grid grid-cols-2 gap-3">
                        <div className="space-y-1">
                          <div className="flex justify-between text-[10px] text-stone-500">
                            <span>ブラシサイズ</span>
                            <span className="font-bold text-pink-500">{eraserSize}px</span>
                          </div>
                          <input
                            type="range"
                            min="5"
                            max="80"
                            value={eraserSize}
                            onChange={(e) => setEraserSize(Number(e.target.value))}
                            className="w-full accent-pink-400 bg-pink-100 rounded-lg appearance-none h-1"
                          />
                        </div>

                        <div className="space-y-1">
                          <div className="flex justify-between text-[10px] text-stone-500">
                            <span>境界ぼかし(ソフト度)</span>
                            <span className="font-bold text-pink-500">{Math.round((1 - eraserHardness) * 100)}%</span>
                          </div>
                          <input
                            type="range"
                            min="0.0"
                            max="0.9"
                            step="0.05"
                            value={eraserHardness}
                            onChange={(e) => setEraserHardness(Number(e.target.value))}
                            className="w-full accent-pink-400 bg-pink-100 rounded-lg appearance-none h-1"
                          />
                        </div>
                      </div>

                      {/* 【重要】透過先カテゴリーの選択 */}
                      <div className="bg-[#fff8f9] p-2.5 rounded-xl border border-pink-100/60 flex items-center justify-between gap-3">
                        <span className="text-[11px] font-bold text-pink-600 shrink-0 flex items-center gap-1">
                          <Tag className="w-3 h-3" /> カテゴリー指定:
                        </span>
                        <div className="flex-1 grid grid-cols-4 gap-1">
                          {categories.map((cat) => (
                            <button
                              key={cat.key}
                              onClick={() => setNewImageCategory(cat.key)}
                              className={`py-1 rounded text-[10px] font-bold transition flex flex-col items-center justify-center ${
                                newImageCategory === cat.key
                                  ? 'bg-pink-400 text-white shadow-sm'
                                  : 'bg-white hover:bg-pink-50/50 text-stone-500 border border-pink-100/40'
                              }`}
                            >
                              <span>{cat.icon}</span>
                              <span className="scale-[0.9] origin-center">{cat.label}</span>
                            </button>
                          ))}
                        </div>
                      </div>

                      <div className="grid grid-cols-2 gap-2">
                        <button
                          onClick={handleAutoColorErase}
                          className="flex items-center justify-center gap-1 py-2 bg-pink-50 hover:bg-pink-100/80 text-[#6b4c4c] text-xs font-extrabold rounded-full border border-pink-200 transition"
                        >
                          <Sparkles className="w-3.5 h-3.5 text-pink-400 fill-pink-100" />
                          <span>一括自動で透過</span>
                        </button>
                        <button
                          onClick={handleAddToCanvas}
                          className="flex items-center justify-center gap-1 py-2 bg-gradient-to-r from-pink-400 to-rose-300 text-white text-xs font-extrabold rounded-full transition shadow-sm animate-pulse"
                        >
                          <Check className="w-3.5 h-3.5" />
                          <span>仕立ててクローゼットへ</span>
                        </button>
                      </div>
                    </div>
                  )}
                </div>
              ) : (
                // === 【新機能】仕立て済みお洋服一覧 タブ（カテゴリー切り替え可能） ===
                <div className="flex-1 flex flex-col justify-between h-full">
                  
                  {/* カテゴリー絞り込み用サブタブ */}
                  <div className="bg-[#fff9fa] p-1.5 rounded-xl border border-pink-100/50 mb-3 flex gap-1 font-sans justify-around">
                    <button
                      onClick={() => setSelectedCategoryFilter('all')}
                      className={`px-2 py-1 text-[10px] font-bold rounded-lg transition ${
                        selectedCategoryFilter === 'all'
                          ? 'bg-pink-400 text-white shadow-sm'
                          : 'text-stone-500 hover:text-pink-500'
                      }`}
                    >
                      すべて
                    </button>
                    {categories.map((cat) => (
                      <button
                        key={cat.key}
                        onClick={() => setSelectedCategoryFilter(cat.key)}
                        className={`px-2 py-1 text-[10px] font-bold rounded-lg transition flex items-center gap-0.5 ${
                          selectedCategoryFilter === cat.key
                            ? 'bg-pink-400 text-white shadow-sm'
                            : 'text-stone-500 hover:text-pink-500'
                        }`}
                      >
                        <span>{cat.icon}</span>
                        <span>{cat.label}</span>
                      </button>
                    ))}
                  </div>

                  {/* スクロール可能なコレクションリスト */}
                  <div className="flex-1 overflow-y-auto pr-1 space-y-2.5 max-h-[390px]">
                    {filteredCollection.length === 0 ? (
                      <div className="text-center py-16 text-xs text-stone-400 font-sans">
                        <FolderHeart className="w-8 h-8 mx-auto text-pink-200 mb-2" />
                        このカテゴリーのお洋服はまだありません。<br/>
                        お好みの衣装を新調しましょう🎀
                      </div>
                    ) : (
                      <div className="grid grid-cols-2 gap-2">
                        {filteredCollection.map((item) => {
                          const catObj = categories.find(c => c.key === item.category);
                          return (
                            <div 
                              key={item.id}
                              className="bg-[#fffbfc] border border-pink-100 hover:border-pink-300 rounded-xl p-2 flex flex-col justify-between shadow-sm relative group transition duration-150"
                            >
                              {/* アイテムサムネイル */}
                              <div className="w-full h-24 bg-[#faf5f6] rounded-lg flex items-center justify-center overflow-hidden mb-2 relative">
                                <img 
                                  src={item.src} 
                                  alt={item.name} 
                                  className="max-w-full max-h-full object-contain"
                                />
                                {/* カテゴリーバッジ */}
                                {catObj && (
                                  <span className="absolute top-1 left-1 bg-white/90 border border-pink-100 rounded-full px-1.5 py-0.2 text-[9px] text-pink-600 font-bold scale-[0.85] origin-top-left">
                                    {catObj.icon} {catObj.label}
                                  </span>
                                )}
                              </div>

                              {/* 名前変更（インプット風） */}
                              <input
                                type="text"
                                value={item.name}
                                onChange={(e) => handleRenameCollectionItem(item.id, e.target.value)}
                                className="text-[10px] font-sans font-bold text-stone-700 bg-transparent hover:bg-pink-50/50 border border-transparent hover:border-pink-100 rounded px-1 py-0.5 mb-1.5 focus:outline-none focus:bg-white focus:border-pink-300"
                                placeholder="お洋服の名前を入力"
                              />

                              {/* ボタン */}
                              <div className="flex gap-1 justify-between items-center text-[10px] font-sans">
                                <button
                                  onClick={() => handlePutOnFromCollection(item)}
                                  className="flex-1 py-1.5 bg-pink-100 hover:bg-pink-200 text-pink-700 font-bold rounded-md text-center transition flex items-center justify-center gap-0.5"
                                >
                                  <Plus className="w-3 h-3" />
                                  <span>着せる</span>
                                </button>
                                
                                <button
                                  onClick={(e) => handleDeleteFromCollection(item.id, e)}
                                  className="p-1 hover:bg-rose-50 text-stone-400 hover:text-rose-500 rounded transition"
                                  title="完全削除"
                                >
                                  <Trash2 className="w-3.5 h-3.5" />
                                </button>
                              </div>
                            </div>
                          );
                        })}
                      </div>
                    )}
                  </div>

                  <div className="pt-2 border-t border-pink-50 text-center font-sans">
                    <p className="text-[10px] text-stone-400">
                      ※ストックしたパーツは、クローゼットでいつでも再利用できます。
                    </p>
                  </div>
                </div>
              )}
            </div>
          </div>

          {/* クローゼット内アイテム調整パネル */}
          <div className="bg-white rounded-3xl border-2 border-pink-100 p-5 shadow-md">
            <div className="flex items-center gap-2 mb-3">
              <Layers className="w-4 h-4 text-pink-400" />
              <h3 className="font-extrabold text-[#5c3e3e] text-sm tracking-wider">ドレスコーディネート操作</h3>
            </div>
            
            {selectedImageId ? (
              <div className="space-y-4 font-sans">
                <div className="text-[11px] text-stone-500 flex items-center justify-between bg-pink-50/40 p-2 rounded-lg">
                  <span className="truncate pr-2 font-bold text-pink-600">
                    👗 選択中: {images.find(img => img.id === selectedImageId)?.name || 'お洋服'}
                  </span>
                  <button 
                    onClick={handleDeleteSelected}
                    className="text-rose-500 hover:text-rose-400 font-bold flex items-center gap-0.5 text-xs transition whitespace-nowrap"
                  >
                    <Trash2 className="w-3 h-3" />
                    お片付け
                  </button>
                </div>

                <div className="grid grid-cols-2 gap-2.5 text-xs">
                  <div className="space-y-1">
                    <span className="text-stone-400 text-[10px]">お洋服のサイズ</span>
                    <div className="flex gap-1.5">
                      <button 
                        onClick={() => handleTransformSelected('resize', -15)}
                        className="flex-1 py-1.5 bg-[#fff8f9] hover:bg-pink-50 border border-pink-100 rounded-full font-bold text-center text-pink-600 transition"
                      >
                        小さく (-)
                      </button>
                      <button 
                        onClick={() => handleTransformSelected('resize', 15)}
                        className="flex-1 py-1.5 bg-[#fff8f9] hover:bg-pink-50 border border-pink-100 rounded-full font-bold text-center text-pink-600 transition"
                      >
                        大きく (+)
                      </button>
                    </div>
                  </div>

                  <div className="space-y-1">
                    <span className="text-stone-400 text-[10px]">ふんわり傾ける</span>
                    <div className="flex gap-1.5">
                      <button 
                        onClick={() => handleTransformSelected('rotate', -10)}
                        className="flex-1 py-1.5 bg-[#fff8f9] hover:bg-pink-50 border border-pink-100 rounded-full font-bold text-center text-pink-600 transition"
                      >
                        左傾げ
                      </button>
                      <button 
                        onClick={() => handleTransformSelected('rotate', 10)}
                        className="flex-1 py-1.5 bg-[#fff8f9] hover:bg-pink-50 border border-pink-100 rounded-full font-bold text-center text-pink-600 transition"
                      >
                        右傾げ
                      </button>
                    </div>
                  </div>
                </div>

                <div className="space-y-1">
                  <span className="text-stone-400 text-[10px] block">お洋服の重ね順（アウターやインナーの調整に）</span>
                  <div className="flex gap-1.5 text-xs">
                    <button 
                      onClick={() => handleTransformSelected('layer', -1)}
                      className="flex-1 py-2 bg-stone-100 hover:bg-stone-200 text-stone-600 rounded-full font-bold transition"
                    >
                      奥へ（インナーに）
                    </button>
                    <button 
                      onClick={() => handleTransformSelected('layer', 1)}
                      className="flex-1 py-2 bg-[#fbe7eb] hover:bg-[#fadce2] text-pink-700 rounded-full font-bold transition"
                    >
                      手前へ（アウターに）
                    </button>
                  </div>
                </div>
              </div>
            ) : (
              <div className="text-center py-5 text-xs text-stone-400 leading-relaxed font-sans">
                右側のクローゼットでお洋服をクリックすると、サイズ変更や前後重ね順のコントロールがここに現れます。
              </div>
            )}
          </div>
        </div>

        {/* 右側：コーディネートクローゼット */}
        <div className="lg:col-span-7 flex flex-col gap-4">
          <div className="bg-white rounded-3xl border-4 border-[#e9cbd1] p-5 flex flex-col shadow-lg relative">
            <div className="absolute top-0 left-0 right-0 h-4 bg-[radial-gradient(circle_at_bottom,transparent_40%,#e9cbd1_40%)] bg-[length:16px_16px] -translate-y-0.5"></div>
            
            <div className="flex flex-wrap items-center justify-between gap-4 mb-4 pb-4 border-b border-pink-50 pt-2">
              <div className="flex items-center gap-2">
                <div className="p-2 bg-pink-50 rounded-full border border-pink-100">
                  <Heart className="w-4 h-4 text-pink-400 fill-pink-200" />
                </div>
                <div>
                  <h2 className="font-black text-[#5c3e3e] text-base tracking-widest">Doll’s Dressing Closet</h2>
                  <p className="text-[11px] text-rose-400 italic">あなただけの甘く可憐なワードローブ</p>
                </div>
              </div>

              {/* キャンバス用コントロール */}
              <div className="flex items-center gap-2 font-sans">
                <button
                  onClick={() => setCanvasZoom(prev => Math.max(0.6, prev - 0.1))}
                  className="p-1.5 bg-[#fff8f9] hover:bg-pink-50 rounded-full border border-pink-100 text-pink-500 transition"
                  title="縮小表示"
                >
                  <ZoomOut className="w-3.5 h-3.5" />
                </button>
                <span className="text-xs font-mono text-[#5c3e3e] font-bold px-1">{Math.round(canvasZoom * 100)}%</span>
                <button
                  onClick={() => setCanvasZoom(prev => Math.min(1.4, prev + 0.1))}
                  className="p-1.5 bg-[#fff8f9] hover:bg-pink-50 rounded-full border border-pink-100 text-pink-500 transition"
                  title="拡大表示"
                >
                  <ZoomIn className="w-3.5 h-3.5" />
                </button>
                <div className="h-4 w-[1px] bg-pink-100 mx-1"></div>
                <button
                  onClick={handleClearCanvas}
                  className="px-3 py-1.5 bg-rose-50 hover:bg-rose-100 text-rose-600 text-xs font-bold rounded-full border border-rose-200 transition"
                >
                  クローゼット全片付け
                </button>
              </div>
            </div>

            {/* コーディネートフィールド */}
            <div className="relative w-full overflow-hidden bg-[#fff9fa] rounded-2xl border-4 border-[#e9cbd1] shadow-inner p-1">
              <div className="absolute inset-2 border border-dashed border-[#e6cfb3] pointer-events-none z-10"></div>
              
              <div 
                ref={coordinateFieldRef}
                onMouseMove={handleCanvasMouseMove}
                onMouseUp={handleCanvasMouseUp}
                onMouseLeave={handleCanvasMouseUp}
                onTouchMove={handleCanvasMouseMove}
                onTouchEnd={handleCanvasMouseUp}
                className="w-full h-[690px] relative overflow-hidden transition-transform duration-75 select-none bg-white bg-[radial-gradient(#fae3e6_1.5px,transparent_1.5px)] [background-size:20px_20px]"
                style={{ 
                  transform: `scale(${canvasZoom})`,
                  transformOrigin: 'top left',
                  width: `${100 / canvasZoom}%`,
                  height: `${690 / canvasZoom}px`,
                  touchAction: 'none'
                }}
              >
                {/* プレースホルダー */}
                {images.length === 0 && (
                  <div className="absolute inset-0 flex flex-col items-center justify-center p-6 text-center pointer-events-none">
                    <div className="p-4 rounded-full bg-pink-50 border border-pink-100 text-pink-300 mb-3 animate-pulse">
                      <Heart className="w-7 h-7 fill-pink-50" />
                    </div>
                    <p className="text-sm font-bold text-stone-600">お洋服がまだ飾られていません</p>
                    <p className="text-xs text-stone-400 mt-1 max-w-xs leading-relaxed font-sans">
                      左の「仕立てた服リスト」タブからお洋服を「着せる」ボタンで配置するか、新しいパーツを仕立ててください。
                    </p>
                  </div>
                )}

                {/* キャンバス上のドールアイテム */}
                {images.map((img) => {
                  const isSelected = img.id === selectedImageId;
                  return (
                    <div
                      key={img.id}
                      onMouseDown={(e) => handleCanvasItemMouseDown(img.id, e)}
                      onTouchStart={(e) => handleCanvasItemMouseDown(img.id, e)}
                      className={`absolute cursor-move select-none p-1 transition-shadow duration-150 ${
                        isSelected ? 'ring-2 ring-pink-400 shadow-xl z-50 bg-pink-50/10 rounded-lg' : 'hover:ring-1 hover:ring-pink-200'
                      }`}
                      style={{
                        left: `${img.x}px`,
                        top: `${img.y}px`,
                        width: `${img.width}px`,
                        height: `${img.height}px`,
                        transform: `rotate(${img.rotation}deg)`,
                        zIndex: img.zIndex,
                        touchAction: 'none'
                      }}
                    >
                      <img
                        src={img.src}
                        alt={img.name}
                        className="w-full h-full object-contain pointer-events-none rounded-lg"
                      />
                      
                      {/* クイック回転＆消去 */}
                      {isSelected && (
                        <div className="absolute -top-3.5 -right-3.5 flex gap-1 bg-white border border-pink-200 rounded-full px-2 py-1 shadow-md pointer-events-auto z-50 animate-fadeIn">
                          <button
                            onClick={(e) => {
                              e.stopPropagation();
                              handleTransformSelected('rotate', 30);
                            }}
                            className="p-1 hover:bg-pink-50 rounded-full text-pink-500 transition"
                            title="30度傾ける"
                          >
                            <RotateCw className="w-3.5 h-3.5" />
                          </button>
                          <button
                            onClick={(e) => {
                              e.stopPropagation();
                              handleDeleteSelected();
                            }}
                            className="p-1 hover:bg-pink-50 rounded-full text-rose-500 transition"
                            title="お片付け"
                          >
                            <Trash2 className="w-3.5 h-3.5" />
                          </button>
                        </div>
                      )}
                    </div>
                  );
                })}
              </div>
            </div>

            {/* ボトム操作案内 */}
            <div className="mt-4 flex flex-wrap items-center justify-between text-xs text-stone-500 gap-2 font-sans">
              <p className="flex items-center gap-1">
                <span className="text-pink-400">🎀</span> 
                アイテムを移動・回転・拡縮して、あなただけの特別なコーデを創り上げてください。
              </p>
              <span className="bg-pink-50/70 px-3 py-1 rounded-full border border-pink-100 text-[#5c3e3e] font-bold">
                クローゼット内: {images.length} アイテム
              </span>
            </div>
          </div>
        </div>

      </main>

      <footer className="border-t border-pink-100 bg-white py-5 text-center text-xs text-rose-400/80">
        <p className="font-serif italic">🎀 Maison de Poupée 〜 Alice's Wardrobe Closet 〜 🎀</p>
      </footer>
    </div>
  );
}
