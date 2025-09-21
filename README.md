// main.dart
import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:google_sign_in/google_sign_in.dart';
import 'package:fl_chart/fl_chart.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';

final FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
    FlutterLocalNotificationsPlugin();

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const MyApp());
}

// ===== MyApp =====
class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Game Tracker',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
      ),
      home: const AuthPage(),
    );
  }
}

// ===== Autenticação com Google =====
class AuthPage extends StatefulWidget {
  const AuthPage({super.key});
  @override
  State<AuthPage> createState() => _AuthPageState();
}

class _AuthPageState extends State<AuthPage> {
  bool loading = false;

  Future<void> signInWithGoogle() async {
    setState(() => loading = true);

    final GoogleSignInAccount? googleUser = await GoogleSignIn().signIn();
    if (googleUser == null) {
      setState(() => loading = false);
      return; // Usuário cancelou
    }

    final GoogleSignInAuthentication googleAuth = await googleUser.authentication;
    final credential = GoogleAuthProvider.credential(
      accessToken: googleAuth.accessToken,
      idToken: googleAuth.idToken,
    );

    await FirebaseAuth.instance.signInWithCredential(credential);
    setState(() => loading = false);

    Navigator.pushReplacement(
      context,
      MaterialPageRoute(builder: (context) => const HomePage()),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Center(
        child: loading
            ? const CircularProgressIndicator()
            : ElevatedButton.icon(
                icon: const Icon(Icons.login),
                label: const Text('Entrar com Google'),
                onPressed: signInWithGoogle,
              ),
      ),
    );
  }
}

// ===== HomePage =====
class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final user = FirebaseAuth.instance.currentUser!;
  final FirebaseFirestore db = FirebaseFirestore.instance;
  List<Map<String, dynamic>> games = [];

  String searchQuery = '';
  String filterStatus = 'Todos';
  String filterPlatform = 'Todos';
  String sortOption = 'Nome';

  @override
  void initState() {
    super.initState();
    loadGames();
  }

  Future<void> loadGames() async {
    final snapshot = await db.collection('users').doc(user.uid).collection('games').get();
    setState(() {
      games = snapshot.docs.map((doc) => doc.data()).toList();
    });
  }

  Future<void> addGame(Map<String, dynamic> game) async {
    final docRef = db.collection('users').doc(user.uid).collection('games').doc();
    game['id'] = docRef.id;
    await docRef.set(game);
    setState(() {
      games.add(game);
    });
  }

  Future<void> updateGame(Map<String, dynamic> game) async {
    await db.collection('users').doc(user.uid).collection('games').doc(game['id']).update(game);
  }

  void toggleFavorite(int index) {
    setState(() {
      games[index]['favorite'] = !(games[index]['favorite'] ?? false);
    });
    updateGame(games[index]);
  }

  List<Map<String, dynamic>> get filteredAndSortedGames {
    final filtered = games.where((g) {
      final matchSearch = g['title'].toLowerCase().contains(searchQuery.toLowerCase());
      final matchStatus = filterStatus == 'Todos' || g['status'] == filterStatus;
      final matchPlatform = filterPlatform == 'Todos' || g['platform'] == filterPlatform;
      return matchSearch && matchStatus && matchPlatform;
    }).toList();

    filtered.sort((a, b) {
      if ((b['favorite'] ?? false) && !(a['favorite'] ?? false)) return 1;
      if ((a['favorite'] ?? false) && !(b['favorite'] ?? false)) return -1;
      switch (sortOption) {
        case 'Data':
          return DateTime.parse(a['addedAt']).compareTo(DateTime.parse(b['addedAt']));
        case 'Tempo':
          return (a['completionTime'] ?? 0).compareTo(b['completionTime'] ?? 0);
        default:
          return a['title'].compareTo(b['title']);
      }
    });

    return filtered;
  }

  @override
  Widget build(BuildContext context) {
    final platforms = ['Todos'] + games.map((g) => g['platform'] as String).toSet().toList();
    final statuses = ['Todos'] + games.map((g) => g['status'] as String).toSet().toList();
    final sortOptions = ['Nome', 'Data', 'Tempo'];

    return Scaffold(
      appBar: AppBar(
        title: const Text('Minha Biblioteca de Jogos'),
        actions: [
          IconButton(
            icon: const Icon(Icons.logout),
            onPressed: () async {
              await FirebaseAuth.instance.signOut();
              await GoogleSignIn().signOut();
              Navigator.pushReplacement(context, MaterialPageRoute(builder: (context) => const AuthPage()));
            },
          ),
          IconButton(
            icon: const Icon(Icons.bar_chart),
            onPressed: () {
              Navigator.push(context, MaterialPageRoute(builder: (context) => StatsPage(games: games)));
            },
          ),
          IconButton(
            icon: const Icon(Icons.new_releases),
            onPressed: () {
              Navigator.push(context, MaterialPageRoute(builder: (context) => const UpcomingGamesPage()));
            },
          ),
          IconButton(
            icon: const Icon(Icons.add),
            onPressed: () async {
              final newGame = await Navigator.push(
                  context,
                  MaterialPageRoute(
                      builder: (context) => AddGamePage(
                            onAdd: addGame,
                          )));
            },
          ),
        ],
      ),
      body: Column(
        children: [
          Padding(
            padding: const EdgeInsets.all(8.0),
            child: TextField(
              decoration: const InputDecoration(
                labelText: 'Pesquisar jogos',
                prefixIcon: Icon(Icons.search),
                border: OutlineInputBorder(),
              ),
              onChanged: (value) => setState(() => searchQuery = value),
            ),
          ),
          Padding(
            padding: const EdgeInsets.symmetric(horizontal: 8.0, vertical: 4.0),
            child: Row(
              children: [
                Expanded(
                  child: DropdownButtonFormField<String>(
                    value: filterStatus,
                    items: statuses.map((s) => DropdownMenuItem(value: s, child: Text(s))).toList(),
                    onChanged: (value) => setState(() => filterStatus = value!),
                    decoration: const InputDecoration(labelText: 'Status', border: OutlineInputBorder()),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: DropdownButtonFormField<String>(
                    value: filterPlatform,
                    items: platforms.map((p) => DropdownMenuItem(value: p, child: Text(p))).toList(),
                    onChanged: (value) => setState(() => filterPlatform = value!),
                    decoration: const InputDecoration(labelText: 'Plataforma', border: OutlineInputBorder()),
                  ),
                ),
                const SizedBox(width: 8),
                Expanded(
                  child: DropdownButtonFormField<String>(
                    value: sortOption,
                    items: sortOptions.map((s) => DropdownMenuItem(value: s, child: Text(s))).toList(),
                    onChanged: (value) => setState(() => sortOption = value!),
                    decoration: const InputDecoration(labelText: 'Ordenar por', border: OutlineInputBorder()),
                  ),
                ),
              ],
            ),
          ),
          Expanded(
            child: filteredAndSortedGames.isEmpty
                ? const Center(child: Text('Nenhum jogo encontrado'))
                : GridView.builder(
                    padding: const EdgeInsets.all(12),
                    gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
                      crossAxisCount: 2,
                      childAspectRatio: 0.75,
                      crossAxisSpacing: 10,
                      mainAxisSpacing: 10,
                    ),
                    itemCount: filteredAndSortedGames.length,
                    itemBuilder: (context, index) {
                      final game = filteredAndSortedGames[index];
                      return Card(
                        shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
                        elevation: 4,
                        child: Padding(
                          padding: const EdgeInsets.all(8.0),
                          child: Column(
                            crossAxisAlignment: CrossAxisAlignment.start,
                            children: [
                              Expanded(
                                child: Container(
                                  decoration: BoxDecoration(
                                    borderRadius: BorderRadius.circular(12),
                                    color: Colors.deepPurple.shade100,
                                  ),
                                  child: const Center(child: Icon(Icons.videogame_asset, size: 50)),
                                ),
                              ),
                              const SizedBox(height: 8),
                              Text(game['title'] ?? '', maxLines: 2, overflow: TextOverflow.ellipsis, style: const TextStyle(fontWeight: FontWeight.bold)),
                              Text(game['platform'] ?? '', style: TextStyle(color: Colors.grey.shade600, fontSize: 12)),
                              Text('Status: ${game['status'] ?? ''}', style: const TextStyle(fontSize: 12)),
                              Align(
                                alignment: Alignment.centerRight,
                                child: IconButton(
                                  icon: Icon(game['favorite'] == true ? Icons.star : Icons.star_border, color: Colors.amber),
                                  onPressed: () => toggleFavorite(games.indexOf(game)),
                                ),
                              ),
                            ],
                          ),
                        ),
                      );
                    },
                  ),
          ),
        ],
      ),
    );
  }
}

// ===== AddGamePage =====
class AddGamePage extends StatefulWidget {
  final Function(Map<String, dynamic>) onAdd;

  const AddGamePage({super.key, required this.onAdd});

  @override
  State<AddGamePage> createState() => _AddGamePageState();
}

class _AddGamePageState extends State<AddGamePage> {
  final _formKey = GlobalKey<FormState>();
  final TextEditingController titleController = TextEditingController();
  final TextEditingController platformController = TextEditingController();
  String status = 'Quero jogar';
  final TextEditingController completionController = TextEditingController();
  final TextEditingController releaseController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Adicionar Jogo')),
      body: Padding(
        padding: const EdgeInsets.all(12.0),
        child: Form(
          key: _formKey,
          child: ListView(
            children: [
              TextFormField(controller: titleController, decoration: const InputDecoration(labelText: 'Título'), validator: (v) => v!.isEmpty ? 'Preencha o título' : null),
              TextFormField(controller: platformController, decoration: const InputDecoration(labelText: 'Plataforma'), validator: (v) => v!.isEmpty ? 'Preencha a plataforma' : null),
              DropdownButtonFormField<String>(
                value: status,
                items: ['Quero jogar', 'Jogando', 'Zerado'].map((s) => DropdownMenuItem(value: s, child: Text(s))).toList(),
                onChanged: (v) => setState(() => status = v!),
                decoration: const InputDecoration(labelText: 'Status'),
              ),
              TextFormField(controller: completionController, decoration: const InputDecoration(labelText: 'Tempo de conclusão (h)'), keyboardType: TextInputType.number),
              TextFormField(controller: releaseController, decoration: const InputDecoration(labelText: 'Data de lançamento (AAAA-MM-DD)')),
              const SizedBox(height: 20),
              ElevatedButton(
                  onPressed: () {
                    if (_formKey.currentState!.validate()) {
                      widget.onAdd({
                        'title': titleController.text,
                        'platform': platformController.text,
                        'status': status,
                        'completionTime': int.tryParse(completionController.text) ?? 0,
                        'releaseDate': releaseController.text,
                        'addedAt': DateTime.now().toIso8601String(),
                        'favorite': false,
                      });
                      Navigator.pop(context);
                    }
                  },
                  child: const Text('Adicionar')),
            ],
          ),
        ),
      ),
    );
  }
}

// ===== UpcomingGamesPage =====
class UpcomingGamesPage extends StatefulWidget {
  const UpcomingGamesPage({super.key});

  @override
  State<UpcomingGamesPage> createState() => _UpcomingGamesPageState();
}

class _UpcomingGamesPageState extends State<UpcomingGamesPage> {
  final user = FirebaseAuth.instance.currentUser!;
  final FirebaseFirestore db = FirebaseFirestore.instance;
  List<Map<String, dynamic>> upcomingGames = [];

  @override
  void initState() {
    super.initState();
    loadUpcomingGames();
  }

  Future<void> loadUpcomingGames() async {
    final now = DateTime.now();
    final snapshot = await db.collection('users').doc(user.uid).collection('games').get();
    setState(() {
      upcomingGames = snapshot.docs
          .map((doc) => doc.data())
          .where((g) {
            final dateStr = g['releaseDate'];
            if (dateStr == null) return false;
            final release = DateTime.tryParse(dateStr);
            return release != null && release.isAfter(now);
          })
          .toList();
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Lançamentos Futuros')),
      body: upcomingGames.isEmpty
          ? const Center(child: Text('Nenhum lançamento futuro'))
          : ListView.builder(
              itemCount: upcomingGames.length,
              itemBuilder: (context, index) {
                final game = upcomingGames[index];
                return ListTile(
                  leading: const Icon(Icons.videogame_asset),
                  title: Text(game['title'] ?? ''),
                  subtitle: Text('Plataforma: ${game['platform'] ?? ''}\nLançamento: ${game['releaseDate'] ?? ''}'),
                );
              },
            ),
    );
  }
}

// ===== StatsPage =====
class StatsPage extends StatelessWidget {
  final List<Map<String, dynamic>> games;

  const StatsPage({super.key, required this.games});

  @override
  Widget build(BuildContext context) {
    final statusCounts = <String, int>{};
    final platformCounts = <String, int>{};

    for (var g in games) {
      final status = g['status'] ?? 'Desconhecido';
      final platform = g['platform'] ?? 'Desconhecido';
      statusCounts[status] = (statusCounts[status] ?? 0) + 1;
      platformCounts[platform] = (platformCounts[platform] ?? 0) + 1;
    }

    return Scaffold(
      appBar: AppBar(title: const Text('Estatísticas')),
      body: Padding(
        padding: const EdgeInsets.all(12.0),
        child: Column(
          children: [
            const Text('Jogos por Status', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 200, child: PieChart(PieChartData(sections: statusCounts.entries.map((e) {
              return PieChartSectionData(value: e.value.toDouble(), title: '${e.key} (${e.value})', radius: 50);
            }).toList()))),
            const SizedBox(height: 20),
            const Text('Jogos por Plataforma', style: TextStyle(fontSize: 18, fontWeight: FontWeight.bold)),
            SizedBox(height: 200, child: PieChart(PieChartData(sections: platformCounts.entries.map((e) {
              return PieChartSectionData(value: e.value.toDouble(), title: '${e.key} (${e.value})', radius: 50);
            }).toList()))),
          ],
        ),
      ),
    );
  }
}
