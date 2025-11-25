import 'dart:async';
import 'package:flutter/material.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:intl/intl.dart';
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path_provider/path_provider.dart';
import 'dart:io';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await NotificationService().init();
  runApp(MyApp());
}

/// SIMPLE MODELS
class Reminder {
  int? id;
  String title;
  String note;
  DateTime dateTime;
  String category;
  bool done;

  Reminder({
    this.id,
    required this.title,
    this.note = '',
    required this.dateTime,
    this.category = 'General',
    this.done = false,
  });

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'title': title,
      'note': note,
      'dateTime': dateTime.toIso8601String(),
      'category': category,
      'done': done ? 1 : 0,
    };
  }

  static Reminder fromMap(Map<String, dynamic> m) {
    return Reminder(
      id: m['id'],
      title: m['title'],
      note: m['note'],
      dateTime: DateTime.parse(m['dateTime']),
      category: m['category'],
      done: m['done'] == 1,
    );
  }
}

class MoodEntry {
  int? id;
  String mood; // 'happy','sad','stressed','neutral'
  String? note;
  DateTime date;

  MoodEntry({this.id, required this.mood, this.note, required this.date});

  Map<String, dynamic> toMap() => {
        'id': id,
        'mood': mood,
        'note': note,
        'date': date.toIso8601String(),
      };

  static MoodEntry fromMap(Map<String, dynamic> m) => MoodEntry(
        id: m['id'],
        mood: m['mood'],
        note: m['note'],
        date: DateTime.parse(m['date']),
      );
}

/// DATABASE HELPER (SQFLITE)
class DBHelper {
  static final DBHelper _instance = DBHelper._internal();
  factory DBHelper() => _instance;
  DBHelper._internal();

  Database? _db;

  Future<Database> get db async {
    if (_db != null) return _db!;
    _db = await initDb();
    return _db!;
  }

  Future<Database> initDb() async {
    final documentsDirectory = await getApplicationDocumentsDirectory();
    final path = join(documentsDirectory.path, "app_data.db");
    return await openDatabase(path, version: 1, onCreate: _onCreate);
  }

  Future _onCreate(Database db, int version) async {
    await db.execute('''
      CREATE TABLE reminders (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT,
        note TEXT,
        dateTime TEXT,
        category TEXT,
        done INTEGER
      )
    ''');
    await db.execute('''
      CREATE TABLE moods (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        mood TEXT,
        note TEXT,
        date TEXT
      )
    ''');
  }

  // REMINDERS CRUD
  Future<int> insertReminder(Reminder r) async {
    final database = await db;
    return await database.insert('reminders', r.toMap());
  }

  Future<int> updateReminder(Reminder r) async {
    final database = await db;
    return await database.update('reminders', r.toMap(),
        where: 'id = ?', whereArgs: [r.id]);
  }

  Future<int> deleteReminder(int id) async {
    final database = await db;
    return await database.delete('reminders', where: 'id = ?', whereArgs: [id]);
  }

  Future<List<Reminder>> getAllReminders() async {
    final database = await db;
    final res = await database.query('reminders', orderBy: 'dateTime ASC');
    return res.map((m) => Reminder.fromMap(m)).toList();
  }

  // MOODS CRUD
  Future<int> insertMood(MoodEntry m) async {
    final database = await db;
    return await database.insert('moods', m.toMap());
  }

  Future<List<MoodEntry>> getMoodsSince(DateTime since) async {
    final database = await db;
    final res = await database.query('moods',
        where: 'date >= ?', whereArgs: [since.toIso8601String()], orderBy: 'date DESC');
    return res.map((m) => MoodEntry.fromMap(m)).toList();
  }

  Future<List<MoodEntry>> getAllMoods() async {
    final database = await db;
    final res = await database.query('moods', orderBy: 'date DESC');
    return res.map((m) => MoodEntry.fromMap(m)).toList();
  }
}

/// NOTIFICATION SERVICE (flutter_local_notifications)
class NotificationService {
  static final NotificationService _instance = NotificationService._internal();
  factory NotificationService() => _instance;
  NotificationService._internal();

  final FlutterLocalNotificationsPlugin flutterLocalNotificationsPlugin =
      FlutterLocalNotificationsPlugin();

  Future<void> init() async {
    const android = AndroidInitializationSettings('@mipmap/ic_launcher');
    final ios = DarwinInitializationSettings();
    final settings = InitializationSettings(android: android, iOS: ios);
    await flutterLocalNotificationsPlugin.initialize(settings);
  }

  Future<void> showScheduledNotification(int id, String title, String body, DateTime dt) async {
    // Note: In production you should use timezone-aware scheduling.
    final tzDate = dt;
    var androidDetails = const AndroidNotificationDetails(
      'reminder_channel',
      'Recordatorios',
      channelDescription: 'Canal para recordatorios',
      importance: Importance.max,
      priority: Priority.high,
    );
    var iosDetails = const DarwinNotificationDetails();

    await flutterLocalNotificationsPlugin.zonedSchedule(
      id,
      title,
      body,
      // Convert to tz DateTime - here we just use local time zone by using tz.TZDateTime.from
      // For simplicity: use DateTime.now() + duration. But flutter_local_notifications recommends timezone package.
      // We'll use the simpler API with DateTimeComponents if repetition needed.
      // Use the following trick: schedule using scheduledDate as DateTime (works on many platforms),
      // but in latest versions it requires tz. We'll convert with .toUtc then schedule with DateTimeComponents — simplified:
      tz.TZDateTime.from(dt, tz.local),
      NotificationDetails(android: androidDetails, iOS: iosDetails),
      androidAllowWhileIdle: true,
      uiLocalNotificationDateInterpretation:
          UILocalNotificationDateInterpretation.absoluteTime,
    );
  }

  Future<void> cancel(int id) async {
    await flutterLocalNotificationsPlugin.cancel(id);
  }

  Future<void> cancelAll() async {
    await flutterLocalNotificationsPlugin.cancelAll();
  }
}

// tz package initialization (required by flutter_local_notifications zonedSchedule)
import 'package:timezone/data/latest.dart' as tz;
import 'package:timezone/timezone.dart' as tz;

/// Initialize timezone data
Future<void> _initTimeZones() async {
  tz.initializeTimeZones();
  final String name = DateTime.now().timeZoneName;
  tz.setLocalLocation(tz.getLocation(tz.local.name));
}

/// UI
class MyApp extends StatefulWidget {
  @override
  State<MyApp> createState() => _MyAppState();
}

class _MyAppState extends State<MyApp> {
  final DBHelper dbh = DBHelper();

  @override
  void initState() {
    super.initState();
    _initAll();
  }

  Future<void> _initAll() async {
    await _initTimeZones();
    await NotificationService().init();
    setState(() {});
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Uni Health Reminder',
      theme: ThemeData(primarySwatch: Colors.indigo),
      home: HomePage(db: dbh),
    );
  }
}

class HomePage extends StatefulWidget {
  final DBHelper db;
  HomePage({required this.db});
  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  int _selectedIndex = 0;
  List<Widget> _pages = [];

  @override
  void initState() {
    super.initState();
    _pages = [
      DashboardPage(db: widget.db),
      RemindersPage(db: widget.db),
      MoodPage(db: widget.db),
      WellbeingPage(),
    ];
  }

  void _onNav(int idx) => setState(() => _selectedIndex = idx);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('Uni Health Reminder'),
      ),
      body: _pages[_selectedIndex],
      bottomNavigationBar: BottomNavigationBar(
        currentIndex: _selectedIndex,
        onTap: _onNav,
        items: const [
          BottomNavigationBarItem(icon: Icon(Icons.home), label: 'Inicio'),
          BottomNavigationBarItem(icon: Icon(Icons.alarm), label: 'Recordatorios'),
          BottomNavigationBarItem(icon: Icon(Icons.mood), label: 'Estado'),
          BottomNavigationBarItem(icon: Icon(Icons.self_improvement), label: 'Bienestar'),
        ],
      ),
    );
  }
}

/// DASHBOARD
class DashboardPage extends StatefulWidget {
  final DBHelper db;
  DashboardPage({required this.db});
  @override
  State<DashboardPage> createState() => _DashboardPageState();
}

class _DashboardPageState extends State<DashboardPage> {
  List<Reminder> upcoming = [];
  MoodEntry? latestMood;

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final all = await widget.db.getAllReminders();
    final moods = await widget.db.getAllMoods();
    setState(() {
      upcoming = all.where((r) => r.dateTime.isAfter(DateTime.now()) && !r.done).take(3).toList();
      if (moods.isNotEmpty) latestMood = moods.first;
    });
  }

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: _load,
      child: ListView(
        padding: EdgeInsets.all(16),
        children: [
          Text('Resumen del día', style: Theme.of(context).textTheme.headline6),
          SizedBox(height: 12),
          Card(
            child: ListTile(
              title: Text(upcoming.isNotEmpty ? upcoming.first.title : 'No hay recordatorios próximos'),
              subtitle: Text(upcoming.isNotEmpty
                  ? DateFormat('EEE, dd MMM yyyy HH:mm').format(upcoming.first.dateTime)
                  : 'Añade recordatorios para mantenerte al día'),
              leading: Icon(Icons.alarm),
            ),
          ),
          SizedBox(height: 16),
          Text('Estado de ánimo reciente', style: Theme.of(context).textTheme.headline6),
          SizedBox(height: 8),
          Card(
            child: ListTile(
              leading: Text(latestMood != null ? _emojiForMood(latestMood!.mood) : '🙂', style: TextStyle(fontSize: 32)),
              title: Text(latestMood != null ? latestMood!.mood.toUpperCase() : 'No registrado'),
              subtitle: Text(latestMood != null ? (latestMood!.note ?? '') : 'Registra cómo te sientes hoy'),
            ),
          ),
          SizedBox(height: 16),
          ElevatedButton.icon(
            onPressed: () => DefaultTabController.of(context)?.animateTo(1),
            icon: Icon(Icons.alarm_add),
            label: Text('Añadir recordatorio'),
          )
        ],
      ),
    );
  }

  String _emojiForMood(String mood) {
    switch (mood) {
      case 'happy':
        return '😄';
      case 'sad':
        return '😢';
      case 'stressed':
        return '😫';
      case 'neutral':
      default:
        return '😐';
    }
  }
}

/// REMINDERS PAGE
class RemindersPage extends StatefulWidget {
  final DBHelper db;
  RemindersPage({required this.db});
  @override
  State<RemindersPage> createState() => _RemindersPageState();
}

class _RemindersPageState extends State<RemindersPage> {
  List<Reminder> reminders = [];
  final _scaffoldKey = GlobalKey<ScaffoldState>();

  @override
  void initState() {
    super.initState();
    _refresh();
  }

  Future<void> _refresh() async {
    final all = await widget.db.getAllReminders();
    setState(() => reminders = all);
  }

  Future<void> _addOrEdit({Reminder? edit}) async {
    final result = await showDialog<Reminder>(
        context: context,
        builder: (context) {
          return ReminderDialog(existing: edit);
        });
    if (result != null) {
      if (result.id == null) {
        final id = await widget.db.insertReminder(result);
        result.id = id;
        // schedule notification
        await NotificationService().showScheduledNotification(
            id,
            'Recordatorio: ${result.title}',
            result.note.isNotEmpty ? result.note : 'Tienes un recordatorio',
            result.dateTime);
      } else {
        await widget.db.updateReminder(result);
        await NotificationService().cancel(result.id!);
        await NotificationService().showScheduledNotification(
            result.id!,
            'Recordatorio: ${result.title}',
            result.note.isNotEmpty ? result.note : 'Tienes un recordatorio',
            result.dateTime);
      }
      await _refresh();
    }
  }

  Future<void> _delete(Reminder r) async {
    if (r.id != null) {
      await widget.db.deleteReminder(r.id!);
      await NotificationService().cancel(r.id!);
      await _refresh();
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      key: _scaffoldKey,
      body: RefreshIndicator(
        onRefresh: _refresh,
        child: ListView.builder(
          padding: EdgeInsets.all(12),
          itemCount: reminders.length,
          itemBuilder: (context, i) {
            final r = reminders[i];
            return Card(
              child: ListTile(
                title: Text(r.title, style: TextStyle(decoration: r.done ? TextDecoration.lineThrough : null)),
                subtitle: Text('${DateFormat('dd MMM yyyy HH:mm').format(r.dateTime)} • ${r.category}'),
                trailing: PopupMenuButton<String>(
                  onSelected: (v) async {
                    if (v == 'edit') await _addOrEdit(edit: r);
                    if (v == 'delete') await _delete(r);
                    if (v == 'toggle') {
                      r.done = !r.done;
                      await widget.db.updateReminder(r);
                      setState(() {});
                    }
                  },
                  itemBuilder: (_) => [
                    PopupMenuItem(value: 'toggle', child: Text(r.done ? 'Marcar como pendiente' : 'Marcar como hecho')),
                    PopupMenuItem(value: 'edit', child: Text('Editar')),
                    PopupMenuItem(value: 'delete', child: Text('Eliminar')),
                  ],
                ),
                onTap: () => _addOrEdit(edit: r),
              ),
            );
          },
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _addOrEdit(),
        child: Icon(Icons.add),
      ),
    );
  }
}

/// Reminder dialog
class ReminderDialog extends StatefulWidget {
  final Reminder? existing;
  ReminderDialog({this.existing});
  @override
  State<ReminderDialog> createState() => _ReminderDialogState();
}

class _ReminderDialogState extends State<ReminderDialog> {
  final _formKey = GlobalKey<FormState>();
  late TextEditingController _titleC;
  late TextEditingController _noteC;
  DateTime _selected = DateTime.now().add(Duration(hours: 1));
  String _category = 'Academics';

  @override
  void initState() {
    super.initState();
    _titleC = TextEditingController(text: widget.existing?.title ?? '');
    _noteC = TextEditingController(text: widget.existing?.note ?? '');
    if (widget.existing != null) {
      _selected = widget.existing!.dateTime;
      _category = widget.existing!.category;
    }
  }

  @override
  Widget build(BuildContext context) {
    return AlertDialog(
      title: Text(widget.existing == null ? 'Nuevo recordatorio' : 'Editar recordatorio'),
      content: SingleChildScrollView(
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              TextFormField(controller: _titleC, decoration: InputDecoration(labelText: 'Título'), validator: (v) => v == null || v.isEmpty ? 'Requerido' : null),
              TextFormField(controller: _noteC, decoration: InputDecoration(labelText: 'Nota (opcional)')),
              SizedBox(height: 8),
              Row(
                children: [
                  Text('Fecha: ${DateFormat('dd MMM yyyy HH:mm').format(_selected)}'),
                  Spacer(),
                  TextButton(onPressed: _pickDateTime, child: Text('Cambiar')),
                ],
              ),
              DropdownButtonFormField<String>(
                value: _category,
                items: ['Academics', 'Salud', 'Personal', 'General']
                    .map((e) => DropdownMenuItem(value: e, child: Text(e)))
                    .toList(),
                onChanged: (v) => setState(() => _category = v ?? 'General'),
                decoration: InputDecoration(labelText: 'Categoría'),
              ),
            ],
          ),
        ),
      ),
      actions: [
        TextButton(onPressed: () => Navigator.pop(context), child: Text('Cancelar')),
        ElevatedButton(
            onPressed: () {
              if (_formKey.currentState!.validate()) {
                final rem = Reminder(
                  id: widget.existing?.id,
                  title: _titleC.text.trim(),
                  note: _noteC.text.trim(),
                  dateTime: _selected,
                  category: _category,
                  done: widget.existing?.done ?? false,
                );
                Navigator.pop(context, rem);
              }
            },
            child: Text('Guardar'))
      ],
    );
  }

  Future<void> _pickDateTime() async {
    final date = await showDatePicker(context: context, initialDate: _selected, firstDate: DateTime.now(), lastDate: DateTime(2100));
    if (date == null) return;
    final time = await showTimePicker(context: context, initialTime: TimeOfDay.fromDateTime(_selected));
    if (time == null) return;
    setState(() {
      _selected = DateTime(date.year, date.month, date.day, time.hour, time.minute);
    });
  }
}

/// MOOD TRACKER
class MoodPage extends StatefulWidget {
  final DBHelper db;
  MoodPage({required this.db});
  @override
  State<MoodPage> createState() => _MoodPageState();
}

class _MoodPageState extends State<MoodPage> {
  String? _selectedMood;
  TextEditingController _noteC = TextEditingController();
  List<MoodEntry> recent = [];

  @override
  void initState() {
    super.initState();
    _load();
  }

  Future<void> _load() async {
    final list = await widget.db.getAllMoods();
    setState(() => recent = list);
  }

  Future<void> _saveMood() async {
    if (_selectedMood == null) return;
    final entry = MoodEntry(mood: _selectedMood!, note: _noteC.text.trim(), date: DateTime.now());
    await widget.db.insertMood(entry);
    _noteC.clear();
    _selectedMood = null;
    await _load();
  }

  @override
  Widget build(BuildContext context) {
    return ListView(
      padding: EdgeInsets.all(12),
      children: [
        Text('¿Cómo te sientes hoy?', style: Theme.of(context).textTheme.headline6),
        SizedBox(height: 12),
        Wrap(
          spacing: 12,
          children: [
            _moodChip('happy', '😄'),
            _moodChip('neutral', '😐'),
            _moodChip('stressed', '😫'),
            _moodChip('sad', '😢'),
          ],
        ),
        SizedBox(height: 12),
        TextField(controller: _noteC, decoration: InputDecoration(labelText: 'Nota (opcional)')),
        SizedBox(height: 8),
        ElevatedButton(onPressed: _saveMood, child: Text('Registrar')),
        SizedBox(height: 16),
        Text('Entradas recientes', style: Theme.of(context).textTheme.headline6),
        ...recent.map((m) => Card(
              child: ListTile(
                leading: Text(_emojiForMood(m.mood), style: TextStyle(fontSize: 28)),
                title: Text(m.mood),
                subtitle: Text('${DateFormat('dd MMM yyyy HH:mm').format(m.date)}\n${m.note ?? ''}'),
                isThreeLine: true,
              ),
            )),
      ],
    );
  }

  Widget _moodChip(String mood, String emoji) {
    final selected = _selectedMood == mood;
    return ChoiceChip(
      label: Text(emoji, style: TextStyle(fontSize: 24)),
      selected: selected,
      onSelected: (_) => setState(() => _selectedMood = selected ? null : mood),
    );
  }

  String _emojiForMood(String mood) {
    switch (mood) {
      case 'happy':
        return '😄';
      case 'sad':
        return '😢';
      case 'stressed':
        return '😫';
      default:
        return '😐';
    }
  }
}

/// WELLBEING / TIPS
class WellbeingPage extends StatelessWidget {
  final List<Map<String, String>> tips = const [
    {'title': 'Respiración 4-4-4', 'content': 'Inhala 4s, mantén 4s, exhala 4s. Repite 4 veces.'},
    {'title': 'Pausa de 5 minutos', 'content': 'Levántate y estira, camina 2 minutos, hidrátate.'},
    {'title': 'Técnica Pomodoro', 'content': 'Estudia 25 minutos, descansa 5 minutos. Repite 4 veces.'},
    {'title': 'Diario breve', 'content': 'Escribe 3 cosas positivas del día.'},
  ];

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      padding: EdgeInsets.all(12),
      itemCount: tips.length,
      itemBuilder: (context, i) {
        final t = tips[i];
        return Card(
          child: ListTile(
            title: Text(t['title']!),
            subtitle: Text(t['content']!),
          ),
        );
      },
    );
  }
}
