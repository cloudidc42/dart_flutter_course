# Part 81: Advanced Forms
## ขั้นตอนที่ 3121-3160

## 🎯 เป้าหมายของ Part นี้
- เรียนรู้ reactive_forms package สำหรับจัดการฟอร์มแบบ reactive
- สร้าง complex form validation ทั้ง async validators และ cross-field validation
- จัดการ dynamic form fields (เพิ่ม/ลบ fields แบบ runtime)
- สร้าง Form Wizard / Multi-step form
- บันทึกและกู้คืน form state

---

## ขั้นตอนที่ 3121: ติดตั้งและตั้งค่า reactive_forms

```yaml
# pubspec.yaml
name: advanced_forms_demo
description: Advanced Forms with reactive_forms package

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  reactive_forms: ^17.0.1
  shared_preferences: ^2.2.2
  http: ^1.1.0

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

```dart
// lib/main.dart
import 'package:flutter/material.dart';
import 'package:reactive_forms/reactive_forms.dart';
import 'screens/registration_form_screen.dart';
import 'screens/dynamic_form_screen.dart';
import 'screens/form_wizard_screen.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Advanced Forms Demo',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.deepPurple),
        useMaterial3: true,
        inputDecorationTheme: const InputDecorationTheme(
          border: OutlineInputBorder(),
          filled: true,
        ),
      ),
      home: const HomeScreen(),
    );
  }
}

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Advanced Forms'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: ListView(
        padding: const EdgeInsets.all(16),
        children: [
          _buildCard(
            context,
            'Registration Form',
            'Async validation + cross-field validation',
            Icons.person_add,
            () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const RegistrationFormScreen()),
            ),
          ),
          const SizedBox(height: 12),
          _buildCard(
            context,
            'Dynamic Form',
            'Add/remove fields dynamically with FormArray',
            Icons.dynamic_form,
            () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const DynamicFormScreen()),
            ),
          ),
          const SizedBox(height: 12),
          _buildCard(
            context,
            'Form Wizard',
            'Multi-step form with state persistence',
            Icons.smart_button,
            () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const FormWizardScreen()),
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildCard(
    BuildContext context,
    String title,
    String subtitle,
    IconData icon,
    VoidCallback onTap,
  ) {
    return Card(
      elevation: 2,
      child: ListTile(
        leading: Icon(icon, size: 40, color: Theme.of(context).colorScheme.primary),
        title: Text(title, style: const TextStyle(fontWeight: FontWeight.bold)),
        subtitle: Text(subtitle),
        trailing: const Icon(Icons.arrow_forward_ios),
        onTap: onTap,
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3122: สร้าง FormGroup และ FormControl พื้นฐาน

```dart
// lib/screens/registration_form_screen.dart
import 'package:flutter/material.dart';
import 'package:reactive_forms/reactive_forms.dart';
import '../validators/custom_validators.dart';
import '../services/user_service.dart';

class RegistrationFormScreen extends StatefulWidget {
  const RegistrationFormScreen({super.key});

  @override
  State<RegistrationFormScreen> createState() => _RegistrationFormScreenState();
}

class _RegistrationFormScreenState extends State<RegistrationFormScreen> {
  late final FormGroup form;
  bool _isSubmitting = false;

  @override
  void initState() {
    super.initState();
    form = _buildForm();
  }

  FormGroup _buildForm() {
    return FormGroup(
      {
        'username': FormControl<String>(
          value: '',
          validators: [
            Validators.required,
            Validators.minLength(4),
            Validators.maxLength(20),
            Validators.pattern(r'^[a-zA-Z0-9_]+$'),
          ],
          asyncValidators: [UsernameAvailableValidator()],
          asyncValidatorsDebounceTime: 600,
        ),
        'email': FormControl<String>(
          value: '',
          validators: [
            Validators.required,
            Validators.email,
          ],
          asyncValidators: [EmailAvailableValidator()],
          asyncValidatorsDebounceTime: 600,
        ),
        'password': FormControl<String>(
          value: '',
          validators: [
            Validators.required,
            Validators.minLength(8),
            CustomValidators.strongPassword,
          ],
        ),
        'confirmPassword': FormControl<String>(
          value: '',
          validators: [Validators.required],
        ),
        'age': FormControl<int>(
          validators: [
            Validators.required,
            Validators.min(18),
            Validators.max(120),
          ],
        ),
        'terms': FormControl<bool>(
          value: false,
          validators: [Validators.requiredTrue],
        ),
      },
      validators: [CustomValidators.passwordsMatch],
    );
  }

  @override
  void dispose() {
    form.dispose();
    super.dispose();
  }

  Future<void> _onSubmit() async {
    if (form.invalid) {
      form.markAllAsTouched();
      return;
    }

    setState(() => _isSubmitting = true);

    try {
      await Future.delayed(const Duration(seconds: 2));
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('Registration successful!'),
            backgroundColor: Colors.green,
          ),
        );
        form.reset();
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Error: $e'), backgroundColor: Colors.red),
        );
      }
    } finally {
      if (mounted) setState(() => _isSubmitting = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Registration Form'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
      ),
      body: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: ReactiveForm(
          formGroup: form,
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              _UsernameField(),
              const SizedBox(height: 16),
              _EmailField(),
              const SizedBox(height: 16),
              _PasswordField(),
              const SizedBox(height: 16),
              _ConfirmPasswordField(),
              const SizedBox(height: 16),
              _AgeField(),
              const SizedBox(height: 16),
              _TermsField(),
              const SizedBox(height: 24),
              ReactiveFormConsumer(
                builder: (context, form, child) {
                  return ElevatedButton(
                    onPressed: _isSubmitting
                        ? null
                        : (form.valid ? _onSubmit : null),
                    style: ElevatedButton.styleFrom(
                      padding: const EdgeInsets.all(16),
                    ),
                    child: _isSubmitting
                        ? const CircularProgressIndicator()
                        : const Text('Register', style: TextStyle(fontSize: 16)),
                  );
                },
              ),
              const SizedBox(height: 12),
              OutlinedButton(
                onPressed: () => form.reset(),
                child: const Text('Reset Form'),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class _UsernameField extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ReactiveTextField<String>(
      formControlName: 'username',
      decoration: const InputDecoration(
        labelText: 'Username',
        hintText: 'Enter username (4-20 chars, alphanumeric)',
        prefixIcon: Icon(Icons.person),
      ),
      validationMessages: {
        ValidationMessage.required: (_) => 'Username is required',
        ValidationMessage.minLength: (e) =>
            'Minimum ${(e as Map)['requiredLength']} characters',
        ValidationMessage.maxLength: (e) =>
            'Maximum ${(e as Map)['requiredLength']} characters',
        ValidationMessage.pattern: (_) =>
            'Only letters, numbers, and underscores',
        'usernameAvailable': (_) => 'Username is already taken',
      },
      showErrors: (control) => control.invalid && control.touched && !control.pending,
      suffix: ReactiveStatusListenableBuilder(
        formControlName: 'username',
        builder: (context, control, child) {
          if (control.pending) return const CircularProgressIndicator();
          if (control.valid && control.value?.isNotEmpty == true) {
            return const Icon(Icons.check_circle, color: Colors.green);
          }
          return const SizedBox.shrink();
        },
      ),
    );
  }
}

class _EmailField extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ReactiveTextField<String>(
      formControlName: 'email',
      keyboardType: TextInputType.emailAddress,
      decoration: const InputDecoration(
        labelText: 'Email',
        hintText: 'Enter your email address',
        prefixIcon: Icon(Icons.email),
      ),
      validationMessages: {
        ValidationMessage.required: (_) => 'Email is required',
        ValidationMessage.email: (_) => 'Enter a valid email address',
        'emailAvailable': (_) => 'Email is already registered',
      },
      showErrors: (control) => control.invalid && control.touched && !control.pending,
    );
  }
}

class _PasswordField extends StatefulWidget {
  @override
  State<_PasswordField> createState() => _PasswordFieldState();
}

class _PasswordFieldState extends State<_PasswordField> {
  bool _obscureText = true;

  @override
  Widget build(BuildContext context) {
    return ReactiveTextField<String>(
      formControlName: 'password',
      obscureText: _obscureText,
      decoration: InputDecoration(
        labelText: 'Password',
        hintText: 'Min 8 chars, mixed case, number, symbol',
        prefixIcon: const Icon(Icons.lock),
        suffixIcon: IconButton(
          icon: Icon(_obscureText ? Icons.visibility : Icons.visibility_off),
          onPressed: () => setState(() => _obscureText = !_obscureText),
        ),
      ),
      validationMessages: {
        ValidationMessage.required: (_) => 'Password is required',
        ValidationMessage.minLength: (_) => 'Minimum 8 characters',
        'strongPassword': (_) =>
            'Must contain uppercase, lowercase, number, and special character',
      },
      showErrors: (control) => control.invalid && control.touched,
    );
  }
}

class _ConfirmPasswordField extends StatefulWidget {
  @override
  State<_ConfirmPasswordField> createState() => _ConfirmPasswordFieldState();
}

class _ConfirmPasswordFieldState extends State<_ConfirmPasswordField> {
  bool _obscureText = true;

  @override
  Widget build(BuildContext context) {
    return ReactiveTextField<String>(
      formControlName: 'confirmPassword',
      obscureText: _obscureText,
      decoration: InputDecoration(
        labelText: 'Confirm Password',
        prefixIcon: const Icon(Icons.lock_outline),
        suffixIcon: IconButton(
          icon: Icon(_obscureText ? Icons.visibility : Icons.visibility_off),
          onPressed: () => setState(() => _obscureText = !_obscureText),
        ),
      ),
      validationMessages: {
        ValidationMessage.required: (_) => 'Please confirm your password',
        'passwordsMatch': (_) => 'Passwords do not match',
      },
      showErrors: (control) {
        final form = ReactiveForm.of(context)!;
        return form.invalid &&
            form.touched &&
            form.hasError('passwordsMatch');
      },
    );
  }
}

class _AgeField extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ReactiveTextField<int>(
      formControlName: 'age',
      keyboardType: TextInputType.number,
      decoration: const InputDecoration(
        labelText: 'Age',
        hintText: 'Must be 18 or older',
        prefixIcon: Icon(Icons.cake),
      ),
      validationMessages: {
        ValidationMessage.required: (_) => 'Age is required',
        ValidationMessage.min: (_) => 'Must be at least 18 years old',
        ValidationMessage.max: (_) => 'Invalid age',
      },
      showErrors: (control) => control.invalid && control.touched,
    );
  }
}

class _TermsField extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ReactiveCheckboxListTile(
      formControlName: 'terms',
      title: const Text('I agree to the Terms and Conditions'),
      controlAffinity: ListTileControlAffinity.leading,
      validationMessages: {
        ValidationMessage.requiredTrue: (_) => 'You must accept the terms',
      },
    );
  }
}
```

---

## ขั้นตอนที่ 3123: สร้าง Custom Validators

```dart
// lib/validators/custom_validators.dart
import 'package:reactive_forms/reactive_forms.dart';

class CustomValidators {
  /// Validator that checks password strength
  static Map<String, dynamic>? strongPassword(AbstractControl<dynamic> control) {
    final value = control.value as String?;
    if (value == null || value.isEmpty) return null;

    final hasUppercase = value.contains(RegExp(r'[A-Z]'));
    final hasLowercase = value.contains(RegExp(r'[a-z]'));
    final hasDigit = value.contains(RegExp(r'[0-9]'));
    final hasSpecial = value.contains(RegExp(r'[!@#$%^&*(),.?":{}|<>]'));

    if (hasUppercase && hasLowercase && hasDigit && hasSpecial) {
      return null;
    }

    return {'strongPassword': true};
  }

  /// Cross-field validator: passwords must match
  static Map<String, dynamic>? passwordsMatch(AbstractControl<dynamic> control) {
    final form = control as FormGroup;
    final password = form.control('password').value as String?;
    final confirmPassword = form.control('confirmPassword').value as String?;

    if (password == null || confirmPassword == null) return null;
    if (password == confirmPassword) return null;

    form.control('confirmPassword').setErrors({'passwordsMatch': true});
    return {'passwordsMatch': true};
  }

  /// Validator for phone number format
  static Map<String, dynamic>? phoneNumber(AbstractControl<dynamic> control) {
    final value = control.value as String?;
    if (value == null || value.isEmpty) return null;

    final phoneRegex = RegExp(r'^\+?[0-9]{10,15}$');
    if (phoneRegex.hasMatch(value)) return null;

    return {'phoneNumber': true};
  }

  /// Range validator
  static ValidatorFunction range(double min, double max) {
    return (AbstractControl<dynamic> control) {
      final value = control.value;
      if (value == null) return null;

      double? numValue;
      if (value is num) {
        numValue = value.toDouble();
      } else if (value is String) {
        numValue = double.tryParse(value);
      }

      if (numValue == null) return {'range': true};
      if (numValue >= min && numValue <= max) return null;

      return {
        'range': {'min': min, 'max': max, 'actual': numValue}
      };
    };
  }

  /// URL validator
  static Map<String, dynamic>? url(AbstractControl<dynamic> control) {
    final value = control.value as String?;
    if (value == null || value.isEmpty) return null;

    try {
      final uri = Uri.parse(value);
      if (uri.hasScheme && (uri.scheme == 'http' || uri.scheme == 'https')) {
        return null;
      }
    } catch (_) {}

    return {'url': true};
  }

  /// Date not in past validator
  static Map<String, dynamic>? notInPast(AbstractControl<dynamic> control) {
    final value = control.value as DateTime?;
    if (value == null) return null;

    if (value.isAfter(DateTime.now())) return null;

    return {'notInPast': true};
  }
}
```

---

## ขั้นตอนที่ 3124: สร้าง Async Validators

```dart
// lib/services/user_service.dart
import 'package:reactive_forms/reactive_forms.dart';

/// Simulated async validator for username availability
class UsernameAvailableValidator extends AsyncValidator<dynamic> {
  // Simulated taken usernames
  static const _takenUsernames = ['admin', 'user123', 'flutter_dev', 'dart_pro'];

  @override
  Future<Map<String, dynamic>?> validate(AbstractControl<dynamic> control) async {
    final value = control.value as String?;

    if (value == null || value.length < 4) return null;

    // Simulate network call
    await Future.delayed(const Duration(milliseconds: 500));

    final isTaken = _takenUsernames.contains(value.toLowerCase());
    return isTaken ? {'usernameAvailable': true} : null;
  }
}

/// Simulated async validator for email availability
class EmailAvailableValidator extends AsyncValidator<dynamic> {
  static const _takenEmails = [
    'admin@example.com',
    'test@test.com',
    'user@flutter.dev',
  ];

  @override
  Future<Map<String, dynamic>?> validate(AbstractControl<dynamic> control) async {
    final value = control.value as String?;

    if (value == null || !value.contains('@')) return null;

    await Future.delayed(const Duration(milliseconds: 600));

    final isTaken = _takenEmails.contains(value.toLowerCase());
    return isTaken ? {'emailAvailable': true} : null;
  }
}
```

---

## ขั้นตอนที่ 3125: Dynamic Form Fields ด้วย FormArray

```dart
// lib/screens/dynamic_form_screen.dart
import 'package:flutter/material.dart';
import 'package:reactive_forms/reactive_forms.dart';

class DynamicFormScreen extends StatefulWidget {
  const DynamicFormScreen({super.key});

  @override
  State<DynamicFormScreen> createState() => _DynamicFormScreenState();
}

class _DynamicFormScreenState extends State<DynamicFormScreen> {
  late final FormGroup form;

  @override
  void initState() {
    super.initState();
    form = _buildForm();
  }

  FormGroup _buildForm() {
    return FormGroup({
      'projectName': FormControl<String>(
        value: '',
        validators: [Validators.required, Validators.minLength(3)],
      ),
      'description': FormControl<String>(
        value: '',
        validators: [Validators.maxLength(500)],
      ),
      'tags': FormArray<String>([
        FormControl<String>(value: '', validators: [Validators.required]),
      ]),
      'teamMembers': FormArray<Map<String, dynamic>>([
        _buildMemberGroup(),
      ]),
      'milestones': FormArray<Map<String, dynamic>>([]),
    });
  }

  FormGroup _buildMemberGroup([String? name, String? role, String? email]) {
    return FormGroup({
      'name': FormControl<String>(
        value: name ?? '',
        validators: [Validators.required],
      ),
      'role': FormControl<String>(
        value: role ?? 'Developer',
        validators: [Validators.required],
      ),
      'email': FormControl<String>(
        value: email ?? '',
        validators: [Validators.required, Validators.email],
      ),
    });
  }

  FormGroup _buildMilestoneGroup() {
    return FormGroup({
      'title': FormControl<String>(
        value: '',
        validators: [Validators.required],
      ),
      'dueDate': FormControl<DateTime>(
        validators: [Validators.required],
      ),
      'completed': FormControl<bool>(value: false),
    });
  }

  FormArray<String> get tagsArray => form.control('tags') as FormArray<String>;
  FormArray<Map<String, dynamic>> get membersArray =>
      form.control('teamMembers') as FormArray<Map<String, dynamic>>;
  FormArray<Map<String, dynamic>> get milestonesArray =>
      form.control('milestones') as FormArray<Map<String, dynamic>>;

  void _addTag() {
    tagsArray.add(FormControl<String>(value: '', validators: [Validators.required]));
  }

  void _removeTag(int index) {
    if (tagsArray.controls.length > 1) tagsArray.removeAt(index);
  }

  void _addMember() {
    membersArray.add(_buildMemberGroup());
  }

  void _removeMember(int index) {
    if (membersArray.controls.length > 1) membersArray.removeAt(index);
  }

  void _addMilestone() {
    milestonesArray.add(_buildMilestoneGroup());
  }

  void _removeMilestone(int index) {
    milestonesArray.removeAt(index);
  }

  @override
  void dispose() {
    form.dispose();
    super.dispose();
  }

  void _onSubmit() {
    if (form.invalid) {
      form.markAllAsTouched();
      return;
    }

    final data = form.value;
    showDialog(
      context: context,
      builder: (_) => AlertDialog(
        title: const Text('Form Data'),
        content: SingleChildScrollView(
          child: Text(data.toString()),
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('OK'),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Dynamic Form'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        actions: [
          ReactiveFormConsumer(
            builder: (context, form, _) => IconButton(
              icon: const Icon(Icons.save),
              onPressed: form.valid ? _onSubmit : null,
              tooltip: 'Save',
            ),
          ),
        ],
      ),
      body: ReactiveForm(
        formGroup: form,
        child: SingleChildScrollView(
          padding: const EdgeInsets.all(16),
          child: Column(
            crossAxisAlignment: CrossAxisAlignment.stretch,
            children: [
              // Project Details Section
              _SectionHeader(title: 'Project Details', icon: Icons.folder),
              const SizedBox(height: 12),
              ReactiveTextField<String>(
                formControlName: 'projectName',
                decoration: const InputDecoration(
                  labelText: 'Project Name',
                  prefixIcon: Icon(Icons.work),
                ),
                validationMessages: {
                  ValidationMessage.required: (_) => 'Required',
                  ValidationMessage.minLength: (_) => 'Min 3 characters',
                },
              ),
              const SizedBox(height: 12),
              ReactiveTextField<String>(
                formControlName: 'description',
                maxLines: 3,
                decoration: const InputDecoration(
                  labelText: 'Description',
                  prefixIcon: Icon(Icons.description),
                  alignLabelWithHint: true,
                ),
                validationMessages: {
                  ValidationMessage.maxLength: (_) => 'Max 500 characters',
                },
              ),

              // Tags Section
              const SizedBox(height: 24),
              _SectionHeader(
                title: 'Tags',
                icon: Icons.label,
                onAdd: _addTag,
                addTooltip: 'Add Tag',
              ),
              const SizedBox(height: 12),
              ReactiveFormArray<String>(
                formArrayName: 'tags',
                builder: (context, array, child) {
                  return Column(
                    children: List.generate(array.controls.length, (index) {
                      return Padding(
                        padding: const EdgeInsets.only(bottom: 8),
                        child: Row(
                          children: [
                            Expanded(
                              child: ReactiveTextField<String>(
                                formControlName: '$index',
                                decoration: InputDecoration(
                                  labelText: 'Tag ${index + 1}',
                                  prefixIcon: const Icon(Icons.tag),
                                ),
                                validationMessages: {
                                  ValidationMessage.required: (_) => 'Tag cannot be empty',
                                },
                              ),
                            ),
                            const SizedBox(width: 8),
                            IconButton(
                              icon: const Icon(Icons.remove_circle_outline),
                              color: Colors.red,
                              onPressed: array.controls.length > 1
                                  ? () => _removeTag(index)
                                  : null,
                            ),
                          ],
                        ),
                      );
                    }),
                  );
                },
              ),

              // Team Members Section
              const SizedBox(height: 24),
              _SectionHeader(
                title: 'Team Members',
                icon: Icons.group,
                onAdd: _addMember,
                addTooltip: 'Add Member',
              ),
              const SizedBox(height: 12),
              ReactiveFormArray<Map<String, dynamic>>(
                formArrayName: 'teamMembers',
                builder: (context, array, child) {
                  return Column(
                    children: List.generate(array.controls.length, (index) {
                      return _MemberCard(
                        index: index,
                        onRemove: array.controls.length > 1
                            ? () => _removeMember(index)
                            : null,
                      );
                    }),
                  );
                },
              ),

              // Milestones Section
              const SizedBox(height: 24),
              _SectionHeader(
                title: 'Milestones',
                icon: Icons.flag,
                onAdd: _addMilestone,
                addTooltip: 'Add Milestone',
              ),
              const SizedBox(height: 12),
              ReactiveFormArray<Map<String, dynamic>>(
                formArrayName: 'milestones',
                builder: (context, array, child) {
                  if (array.controls.isEmpty) {
                    return const Center(
                      child: Padding(
                        padding: EdgeInsets.all(16),
                        child: Text(
                          'No milestones yet. Tap + to add one.',
                          style: TextStyle(color: Colors.grey),
                        ),
                      ),
                    );
                  }
                  return Column(
                    children: List.generate(array.controls.length, (index) {
                      return _MilestoneCard(
                        index: index,
                        onRemove: () => _removeMilestone(index),
                      );
                    }),
                  );
                },
              ),

              const SizedBox(height: 32),
              ElevatedButton.icon(
                onPressed: _onSubmit,
                icon: const Icon(Icons.save),
                label: const Text('Save Project', style: TextStyle(fontSize: 16)),
                style: ElevatedButton.styleFrom(
                  padding: const EdgeInsets.all(16),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class _SectionHeader extends StatelessWidget {
  final String title;
  final IconData icon;
  final VoidCallback? onAdd;
  final String? addTooltip;

  const _SectionHeader({
    required this.title,
    required this.icon,
    this.onAdd,
    this.addTooltip,
  });

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        Icon(icon, color: Theme.of(context).colorScheme.primary),
        const SizedBox(width: 8),
        Text(
          title,
          style: Theme.of(context).textTheme.titleLarge?.copyWith(
                fontWeight: FontWeight.bold,
                color: Theme.of(context).colorScheme.primary,
              ),
        ),
        const Spacer(),
        if (onAdd != null)
          IconButton(
            icon: const Icon(Icons.add_circle),
            onPressed: onAdd,
            tooltip: addTooltip,
            color: Theme.of(context).colorScheme.primary,
          ),
      ],
    );
  }
}

class _MemberCard extends StatelessWidget {
  final int index;
  final VoidCallback? onRemove;

  const _MemberCard({required this.index, this.onRemove});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 12),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          children: [
            Row(
              children: [
                Text(
                  'Member ${index + 1}',
                  style: const TextStyle(fontWeight: FontWeight.bold),
                ),
                const Spacer(),
                if (onRemove != null)
                  IconButton(
                    icon: const Icon(Icons.delete_outline, color: Colors.red),
                    onPressed: onRemove,
                  ),
              ],
            ),
            ReactiveTextField<String>(
              formControlName: '$index.name',
              decoration: const InputDecoration(
                labelText: 'Name',
                prefixIcon: Icon(Icons.person),
                isDense: true,
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Name is required',
              },
            ),
            const SizedBox(height: 8),
            ReactiveDropdownField<String>(
              formControlName: '$index.role',
              decoration: const InputDecoration(
                labelText: 'Role',
                prefixIcon: Icon(Icons.work),
                isDense: true,
              ),
              items: const [
                DropdownMenuItem(value: 'Developer', child: Text('Developer')),
                DropdownMenuItem(value: 'Designer', child: Text('Designer')),
                DropdownMenuItem(value: 'Manager', child: Text('Manager')),
                DropdownMenuItem(value: 'QA', child: Text('QA Engineer')),
                DropdownMenuItem(value: 'DevOps', child: Text('DevOps')),
              ],
            ),
            const SizedBox(height: 8),
            ReactiveTextField<String>(
              formControlName: '$index.email',
              keyboardType: TextInputType.emailAddress,
              decoration: const InputDecoration(
                labelText: 'Email',
                prefixIcon: Icon(Icons.email),
                isDense: true,
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Email is required',
                ValidationMessage.email: (_) => 'Invalid email',
              },
            ),
          ],
        ),
      ),
    );
  }
}

class _MilestoneCard extends StatelessWidget {
  final int index;
  final VoidCallback onRemove;

  const _MilestoneCard({required this.index, required this.onRemove});

  @override
  Widget build(BuildContext context) {
    return Card(
      margin: const EdgeInsets.only(bottom: 12),
      child: Padding(
        padding: const EdgeInsets.all(12),
        child: Column(
          children: [
            Row(
              children: [
                Text(
                  'Milestone ${index + 1}',
                  style: const TextStyle(fontWeight: FontWeight.bold),
                ),
                const Spacer(),
                IconButton(
                  icon: const Icon(Icons.delete_outline, color: Colors.red),
                  onPressed: onRemove,
                ),
              ],
            ),
            ReactiveTextField<String>(
              formControlName: '$index.title',
              decoration: const InputDecoration(
                labelText: 'Title',
                prefixIcon: Icon(Icons.title),
                isDense: true,
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Title is required',
              },
            ),
            const SizedBox(height: 8),
            ReactiveDatePicker<DateTime>(
              formControlName: '$index.dueDate',
              firstDate: DateTime.now(),
              lastDate: DateTime.now().add(const Duration(days: 365 * 5)),
              builder: (context, picker, child) {
                return ReactiveTextField<DateTime>(
                  formControlName: '$index.dueDate',
                  readOnly: true,
                  decoration: InputDecoration(
                    labelText: 'Due Date',
                    prefixIcon: const Icon(Icons.calendar_today),
                    isDense: true,
                    suffixIcon: IconButton(
                      icon: const Icon(Icons.date_range),
                      onPressed: picker.showPicker,
                    ),
                  ),
                  valueAccessor: DateTimeValueAccessor(),
                  validationMessages: {
                    ValidationMessage.required: (_) => 'Due date is required',
                  },
                );
              },
            ),
            const SizedBox(height: 8),
            ReactiveCheckboxListTile(
              formControlName: '$index.completed',
              title: const Text('Mark as completed'),
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3126: Form Wizard / Multi-step Form

```dart
// lib/screens/form_wizard_screen.dart
import 'package:flutter/material.dart';
import 'package:reactive_forms/reactive_forms.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'dart:convert';

class FormWizardScreen extends StatefulWidget {
  const FormWizardScreen({super.key});

  @override
  State<FormWizardScreen> createState() => _FormWizardScreenState();
}

class _FormWizardScreenState extends State<FormWizardScreen> {
  late final PageController _pageController;
  late final FormGroup _personalForm;
  late final FormGroup _addressForm;
  late final FormGroup _professionalForm;
  late final FormGroup _preferencesForm;

  int _currentStep = 0;
  bool _isLoading = true;

  final List<String> _stepTitles = [
    'Personal Info',
    'Address',
    'Professional',
    'Preferences',
  ];

  @override
  void initState() {
    super.initState();
    _pageController = PageController();
    _initializeForms();
    _loadSavedData();
  }

  void _initializeForms() {
    _personalForm = FormGroup({
      'firstName': FormControl<String>(
        value: '',
        validators: [Validators.required, Validators.minLength(2)],
      ),
      'lastName': FormControl<String>(
        value: '',
        validators: [Validators.required, Validators.minLength(2)],
      ),
      'dateOfBirth': FormControl<DateTime>(
        validators: [Validators.required],
      ),
      'gender': FormControl<String>(
        value: 'male',
        validators: [Validators.required],
      ),
      'phone': FormControl<String>(
        validators: [
          Validators.required,
          Validators.pattern(r'^\+?[0-9]{10,15}$'),
        ],
      ),
    });

    _addressForm = FormGroup({
      'street': FormControl<String>(validators: [Validators.required]),
      'city': FormControl<String>(validators: [Validators.required]),
      'state': FormControl<String>(validators: [Validators.required]),
      'country': FormControl<String>(
        value: 'Thailand',
        validators: [Validators.required],
      ),
      'zipCode': FormControl<String>(
        validators: [
          Validators.required,
          Validators.pattern(r'^\d{5}$'),
        ],
      ),
    });

    _professionalForm = FormGroup({
      'occupation': FormControl<String>(validators: [Validators.required]),
      'company': FormControl<String>(),
      'yearsExperience': FormControl<int>(
        validators: [Validators.required, Validators.min(0), Validators.max(50)],
      ),
      'skills': FormArray<String>([
        FormControl<String>(value: ''),
      ]),
      'linkedIn': FormControl<String>(
        validators: [Validators.pattern(r'^https?://.*linkedin\.com.*')],
      ),
    });

    _preferencesForm = FormGroup({
      'newsletter': FormControl<bool>(value: true),
      'notifications': FormControl<bool>(value: true),
      'theme': FormControl<String>(value: 'system'),
      'language': FormControl<String>(value: 'en'),
      'bio': FormControl<String>(
        validators: [Validators.maxLength(200)],
      ),
    });
  }

  Future<void> _loadSavedData() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final savedData = prefs.getString('wizard_form_data');
      if (savedData != null) {
        final data = json.decode(savedData) as Map<String, dynamic>;
        _applyFormData(data);
      }
    } catch (_) {}

    if (mounted) setState(() => _isLoading = false);
  }

  void _applyFormData(Map<String, dynamic> data) {
    if (data['personal'] != null) {
      final personal = data['personal'] as Map<String, dynamic>;
      _personalForm.control('firstName').value = personal['firstName'] ?? '';
      _personalForm.control('lastName').value = personal['lastName'] ?? '';
      _personalForm.control('gender').value = personal['gender'] ?? 'male';
      _personalForm.control('phone').value = personal['phone'] ?? '';
    }

    if (data['address'] != null) {
      final address = data['address'] as Map<String, dynamic>;
      _addressForm.control('street').value = address['street'] ?? '';
      _addressForm.control('city').value = address['city'] ?? '';
      _addressForm.control('state').value = address['state'] ?? '';
      _addressForm.control('country').value = address['country'] ?? 'Thailand';
      _addressForm.control('zipCode').value = address['zipCode'] ?? '';
    }
  }

  Future<void> _saveProgress() async {
    final data = {
      'personal': {
        'firstName': _personalForm.control('firstName').value,
        'lastName': _personalForm.control('lastName').value,
        'gender': _personalForm.control('gender').value,
        'phone': _personalForm.control('phone').value,
      },
      'address': {
        'street': _addressForm.control('street').value,
        'city': _addressForm.control('city').value,
        'state': _addressForm.control('state').value,
        'country': _addressForm.control('country').value,
        'zipCode': _addressForm.control('zipCode').value,
      },
    };

    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('wizard_form_data', json.encode(data));

    if (mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(
          content: Text('Progress saved!'),
          duration: Duration(seconds: 1),
          behavior: SnackBarBehavior.floating,
        ),
      );
    }
  }

  FormGroup get _currentForm {
    switch (_currentStep) {
      case 0: return _personalForm;
      case 1: return _addressForm;
      case 2: return _professionalForm;
      case 3: return _preferencesForm;
      default: return _personalForm;
    }
  }

  void _nextStep() {
    if (_currentForm.invalid) {
      _currentForm.markAllAsTouched();
      return;
    }

    if (_currentStep < _stepTitles.length - 1) {
      setState(() => _currentStep++);
      _pageController.nextPage(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
      );
      _saveProgress();
    } else {
      _submitForm();
    }
  }

  void _previousStep() {
    if (_currentStep > 0) {
      setState(() => _currentStep--);
      _pageController.previousPage(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
      );
    }
  }

  Future<void> _submitForm() async {
    final allData = {
      'personal': _personalForm.value,
      'address': _addressForm.value,
      'professional': _professionalForm.value,
      'preferences': _preferencesForm.value,
    };

    // Clear saved data on successful submission
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove('wizard_form_data');

    if (mounted) {
      showDialog(
        context: context,
        builder: (_) => AlertDialog(
          title: const Row(
            children: [
              Icon(Icons.check_circle, color: Colors.green),
              SizedBox(width: 8),
              Text('Registration Complete!'),
            ],
          ),
          content: const Text(
            'Your profile has been successfully created.',
          ),
          actions: [
            ElevatedButton(
              onPressed: () {
                Navigator.pop(context);
                Navigator.pop(context);
              },
              child: const Text('Done'),
            ),
          ],
        ),
      );
    }
  }

  @override
  void dispose() {
    _pageController.dispose();
    _personalForm.dispose();
    _addressForm.dispose();
    _professionalForm.dispose();
    _preferencesForm.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    if (_isLoading) {
      return const Scaffold(body: Center(child: CircularProgressIndicator()));
    }

    return Scaffold(
      appBar: AppBar(
        title: Text('Step ${_currentStep + 1}: ${_stepTitles[_currentStep]}'),
        backgroundColor: Theme.of(context).colorScheme.inversePrimary,
        actions: [
          IconButton(
            icon: const Icon(Icons.save_outlined),
            onPressed: _saveProgress,
            tooltip: 'Save Progress',
          ),
        ],
      ),
      body: Column(
        children: [
          // Step Indicator
          _StepIndicator(
            currentStep: _currentStep,
            totalSteps: _stepTitles.length,
            titles: _stepTitles,
          ),

          // Form Pages
          Expanded(
            child: PageView(
              controller: _pageController,
              physics: const NeverScrollableScrollPhysics(),
              children: [
                _PersonalInfoStep(form: _personalForm),
                _AddressStep(form: _addressForm),
                _ProfessionalStep(form: _professionalForm),
                _PreferencesStep(form: _preferencesForm),
              ],
            ),
          ),

          // Navigation Buttons
          _NavigationButtons(
            currentStep: _currentStep,
            totalSteps: _stepTitles.length,
            onNext: _nextStep,
            onBack: _previousStep,
          ),
        ],
      ),
    );
  }
}

class _StepIndicator extends StatelessWidget {
  final int currentStep;
  final int totalSteps;
  final List<String> titles;

  const _StepIndicator({
    required this.currentStep,
    required this.totalSteps,
    required this.titles,
  });

  @override
  Widget build(BuildContext context) {
    final primary = Theme.of(context).colorScheme.primary;

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 12),
      color: Theme.of(context).colorScheme.surfaceVariant.withOpacity(0.3),
      child: Row(
        children: List.generate(totalSteps * 2 - 1, (index) {
          if (index.isOdd) {
            final stepIndex = index ~/ 2;
            final isCompleted = currentStep > stepIndex;
            return Expanded(
              child: Container(
                height: 2,
                color: isCompleted ? primary : Colors.grey.shade300,
              ),
            );
          }

          final stepIndex = index ~/ 2;
          final isCompleted = currentStep > stepIndex;
          final isCurrent = currentStep == stepIndex;

          return Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              AnimatedContainer(
                duration: const Duration(milliseconds: 300),
                width: 32,
                height: 32,
                decoration: BoxDecoration(
                  shape: BoxShape.circle,
                  color: isCompleted || isCurrent ? primary : Colors.grey.shade300,
                ),
                child: Center(
                  child: isCompleted
                      ? const Icon(Icons.check, color: Colors.white, size: 18)
                      : Text(
                          '${stepIndex + 1}',
                          style: TextStyle(
                            color: isCurrent ? Colors.white : Colors.grey,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                ),
              ),
              const SizedBox(height: 4),
              Text(
                titles[stepIndex].split(' ').first,
                style: TextStyle(
                  fontSize: 10,
                  color: isCurrent ? primary : Colors.grey,
                  fontWeight: isCurrent ? FontWeight.bold : null,
                ),
              ),
            ],
          );
        }),
      ),
    );
  }
}

class _NavigationButtons extends StatelessWidget {
  final int currentStep;
  final int totalSteps;
  final VoidCallback onNext;
  final VoidCallback onBack;

  const _NavigationButtons({
    required this.currentStep,
    required this.totalSteps,
    required this.onNext,
    required this.onBack,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Theme.of(context).scaffoldBackgroundColor,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.05),
            blurRadius: 8,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: Row(
        children: [
          if (currentStep > 0)
            Expanded(
              child: OutlinedButton.icon(
                onPressed: onBack,
                icon: const Icon(Icons.arrow_back),
                label: const Text('Back'),
              ),
            ),
          if (currentStep > 0) const SizedBox(width: 12),
          Expanded(
            flex: 2,
            child: ElevatedButton.icon(
              onPressed: onNext,
              icon: Icon(
                currentStep < totalSteps - 1 ? Icons.arrow_forward : Icons.check,
              ),
              label: Text(
                currentStep < totalSteps - 1 ? 'Continue' : 'Submit',
                style: const TextStyle(fontSize: 16),
              ),
              style: ElevatedButton.styleFrom(
                padding: const EdgeInsets.all(16),
              ),
            ),
          ),
        ],
      ),
    );
  }
}

class _PersonalInfoStep extends StatelessWidget {
  final FormGroup form;

  const _PersonalInfoStep({required this.form});

  @override
  Widget build(BuildContext context) {
    return ReactiveForm(
      formGroup: form,
      child: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            Row(
              children: [
                Expanded(
                  child: ReactiveTextField<String>(
                    formControlName: 'firstName',
                    decoration: const InputDecoration(labelText: 'First Name'),
                    validationMessages: {
                      ValidationMessage.required: (_) => 'Required',
                      ValidationMessage.minLength: (_) => 'Min 2 chars',
                    },
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: ReactiveTextField<String>(
                    formControlName: 'lastName',
                    decoration: const InputDecoration(labelText: 'Last Name'),
                    validationMessages: {
                      ValidationMessage.required: (_) => 'Required',
                      ValidationMessage.minLength: (_) => 'Min 2 chars',
                    },
                  ),
                ),
              ],
            ),
            const SizedBox(height: 16),
            ReactiveDatePicker<DateTime>(
              formControlName: 'dateOfBirth',
              firstDate: DateTime(1900),
              lastDate: DateTime.now(),
              builder: (context, picker, child) {
                return ReactiveTextField<DateTime>(
                  formControlName: 'dateOfBirth',
                  readOnly: true,
                  decoration: InputDecoration(
                    labelText: 'Date of Birth',
                    prefixIcon: const Icon(Icons.cake),
                    suffixIcon: IconButton(
                      icon: const Icon(Icons.calendar_today),
                      onPressed: picker.showPicker,
                    ),
                  ),
                  valueAccessor: DateTimeValueAccessor(),
                  validationMessages: {
                    ValidationMessage.required: (_) => 'Required',
                  },
                );
              },
            ),
            const SizedBox(height: 16),
            ReactiveDropdownField<String>(
              formControlName: 'gender',
              decoration: const InputDecoration(
                labelText: 'Gender',
                prefixIcon: Icon(Icons.people),
              ),
              items: const [
                DropdownMenuItem(value: 'male', child: Text('Male')),
                DropdownMenuItem(value: 'female', child: Text('Female')),
                DropdownMenuItem(value: 'other', child: Text('Other')),
                DropdownMenuItem(value: 'prefer_not', child: Text('Prefer not to say')),
              ],
            ),
            const SizedBox(height: 16),
            ReactiveTextField<String>(
              formControlName: 'phone',
              keyboardType: TextInputType.phone,
              decoration: const InputDecoration(
                labelText: 'Phone Number',
                prefixIcon: Icon(Icons.phone),
                hintText: '+66xxxxxxxxxx',
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Required',
                ValidationMessage.pattern: (_) => 'Invalid phone number',
              },
            ),
          ],
        ),
      ),
    );
  }
}

class _AddressStep extends StatelessWidget {
  final FormGroup form;

  const _AddressStep({required this.form});

  @override
  Widget build(BuildContext context) {
    return ReactiveForm(
      formGroup: form,
      child: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            ReactiveTextField<String>(
              formControlName: 'street',
              decoration: const InputDecoration(
                labelText: 'Street Address',
                prefixIcon: Icon(Icons.home),
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Required',
              },
            ),
            const SizedBox(height: 16),
            Row(
              children: [
                Expanded(
                  child: ReactiveTextField<String>(
                    formControlName: 'city',
                    decoration: const InputDecoration(labelText: 'City'),
                    validationMessages: {
                      ValidationMessage.required: (_) => 'Required',
                    },
                  ),
                ),
                const SizedBox(width: 12),
                Expanded(
                  child: ReactiveTextField<String>(
                    formControlName: 'state',
                    decoration: const InputDecoration(labelText: 'Province/State'),
                    validationMessages: {
                      ValidationMessage.required: (_) => 'Required',
                    },
                  ),
                ),
              ],
            ),
            const SizedBox(height: 16),
            ReactiveDropdownField<String>(
              formControlName: 'country',
              decoration: const InputDecoration(
                labelText: 'Country',
                prefixIcon: Icon(Icons.flag),
              ),
              items: const [
                DropdownMenuItem(value: 'Thailand', child: Text('Thailand')),
                DropdownMenuItem(value: 'USA', child: Text('United States')),
                DropdownMenuItem(value: 'UK', child: Text('United Kingdom')),
                DropdownMenuItem(value: 'Japan', child: Text('Japan')),
                DropdownMenuItem(value: 'Singapore', child: Text('Singapore')),
              ],
            ),
            const SizedBox(height: 16),
            ReactiveTextField<String>(
              formControlName: 'zipCode',
              keyboardType: TextInputType.number,
              decoration: const InputDecoration(
                labelText: 'Zip/Postal Code',
                prefixIcon: Icon(Icons.local_post_office),
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Required',
                ValidationMessage.pattern: (_) => 'Must be 5 digits',
              },
            ),
          ],
        ),
      ),
    );
  }
}

class _ProfessionalStep extends StatelessWidget {
  final FormGroup form;

  const _ProfessionalStep({required this.form});

  FormArray<String> get skillsArray => form.control('skills') as FormArray<String>;

  @override
  Widget build(BuildContext context) {
    return ReactiveForm(
      formGroup: form,
      child: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            ReactiveTextField<String>(
              formControlName: 'occupation',
              decoration: const InputDecoration(
                labelText: 'Occupation/Job Title',
                prefixIcon: Icon(Icons.work),
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Required',
              },
            ),
            const SizedBox(height: 16),
            ReactiveTextField<String>(
              formControlName: 'company',
              decoration: const InputDecoration(
                labelText: 'Company (Optional)',
                prefixIcon: Icon(Icons.business),
              ),
            ),
            const SizedBox(height: 16),
            ReactiveTextField<int>(
              formControlName: 'yearsExperience',
              keyboardType: TextInputType.number,
              decoration: const InputDecoration(
                labelText: 'Years of Experience',
                prefixIcon: Icon(Icons.timeline),
              ),
              validationMessages: {
                ValidationMessage.required: (_) => 'Required',
                ValidationMessage.min: (_) => 'Cannot be negative',
                ValidationMessage.max: (_) => 'Invalid value',
              },
            ),
            const SizedBox(height: 16),
            ReactiveTextField<String>(
              formControlName: 'linkedIn',
              keyboardType: TextInputType.url,
              decoration: const InputDecoration(
                labelText: 'LinkedIn URL (Optional)',
                prefixIcon: Icon(Icons.link),
                hintText: 'https://linkedin.com/in/...',
              ),
              validationMessages: {
                ValidationMessage.pattern: (_) => 'Invalid LinkedIn URL',
              },
            ),
            const SizedBox(height: 16),
            Row(
              children: [
                const Text('Skills:', style: TextStyle(fontWeight: FontWeight.bold)),
                const Spacer(),
                TextButton.icon(
                  onPressed: () {
                    skillsArray.add(
                      FormControl<String>(value: '', validators: [Validators.required]),
                    );
                  },
                  icon: const Icon(Icons.add),
                  label: const Text('Add Skill'),
                ),
              ],
            ),
            ReactiveFormArray<String>(
              formArrayName: 'skills',
              builder: (context, array, child) {
                return Column(
                  children: List.generate(array.controls.length, (index) {
                    return Padding(
                      padding: const EdgeInsets.only(bottom: 8),
                      child: Row(
                        children: [
                          Expanded(
                            child: ReactiveTextField<String>(
                              formControlName: '$index',
                              decoration: InputDecoration(
                                labelText: 'Skill ${index + 1}',
                                isDense: true,
                              ),
                            ),
                          ),
                          if (array.controls.length > 1)
                            IconButton(
                              icon: const Icon(Icons.remove_circle, color: Colors.red),
                              onPressed: () => skillsArray.removeAt(index),
                            ),
                        ],
                      ),
                    );
                  }),
                );
              },
            ),
          ],
        ),
      ),
    );
  }
}

class _PreferencesStep extends StatelessWidget {
  final FormGroup form;

  const _PreferencesStep({required this.form});

  @override
  Widget build(BuildContext context) {
    return ReactiveForm(
      formGroup: form,
      child: SingleChildScrollView(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            ReactiveCheckboxListTile(
              formControlName: 'newsletter',
              title: const Text('Subscribe to newsletter'),
              subtitle: const Text('Receive weekly updates and tips'),
            ),
            ReactiveCheckboxListTile(
              formControlName: 'notifications',
              title: const Text('Push Notifications'),
              subtitle: const Text('Get notified about important updates'),
            ),
            const SizedBox(height: 16),
            ReactiveDropdownField<String>(
              formControlName: 'theme',
              decoration: const InputDecoration(
                labelText: 'App Theme',
                prefixIcon: Icon(Icons.palette),
              ),
              items: const [
                DropdownMenuItem(value: 'system', child: Text('System Default')),
                DropdownMenuItem(value: 'light', child: Text('Light')),
                DropdownMenuItem(value: 'dark', child: Text('Dark')),
              ],
            ),
            const SizedBox(height: 16),
            ReactiveDropdownField<String>(
              formControlName: 'language',
              decoration: const InputDecoration(
                labelText: 'Language',
                prefixIcon: Icon(Icons.language),
              ),
              items: const [
                DropdownMenuItem(value: 'en', child: Text('English')),
                DropdownMenuItem(value: 'th', child: Text('Thai')),
                DropdownMenuItem(value: 'ja', child: Text('Japanese')),
                DropdownMenuItem(value: 'zh', child: Text('Chinese')),
              ],
            ),
            const SizedBox(height: 16),
            ReactiveTextField<String>(
              formControlName: 'bio',
              maxLines: 4,
              decoration: const InputDecoration(
                labelText: 'Bio (Optional)',
                hintText: 'Tell us about yourself...',
                alignLabelWithHint: true,
              ),
              validationMessages: {
                ValidationMessage.maxLength: (_) => 'Max 200 characters',
              },
            ),
          ],
        ),
      ),
    );
  }
}
```

---

## ขั้นตอนที่ 3127: Form State Persistence

```dart
// lib/services/form_persistence_service.dart
import 'dart:convert';
import 'package:reactive_forms/reactive_forms.dart';
import 'package:shared_preferences/shared_preferences.dart';

class FormPersistenceService {
  static const String _prefix = 'form_state_';

  /// Save form state to SharedPreferences
  static Future<void> saveForm(String formKey, FormGroup form) async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final data = _serializeFormGroup(form);
      await prefs.setString('$_prefix$formKey', json.encode(data));
    } catch (e) {
      print('Error saving form: $e');
    }
  }

  /// Load form state from SharedPreferences
  static Future<bool> loadForm(String formKey, FormGroup form) async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final savedData = prefs.getString('$_prefix$formKey');
      if (savedData == null) return false;

      final data = json.decode(savedData) as Map<String, dynamic>;
      _applyToFormGroup(form, data);
      return true;
    } catch (e) {
      print('Error loading form: $e');
      return false;
    }
  }

  /// Clear saved form state
  static Future<void> clearForm(String formKey) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.remove('$_prefix$formKey');
  }

  /// Check if saved form exists
  static Future<bool> hasSavedForm(String formKey) async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.containsKey('$_prefix$formKey');
  }

  static Map<String, dynamic> _serializeFormGroup(FormGroup group) {
    final result = <String, dynamic>{};

    for (final entry in group.controls.entries) {
      final control = entry.value;

      if (control is FormGroup) {
        result[entry.key] = _serializeFormGroup(control);
      } else if (control is FormArray) {
        result[entry.key] = _serializeFormArray(control);
      } else if (control is FormControl) {
        final value = control.value;
        if (value is DateTime) {
          result[entry.key] = {'_type': 'DateTime', 'value': value.toIso8601String()};
        } else {
          result[entry.key] = value;
        }
      }
    }

    return result;
  }

  static List<dynamic> _serializeFormArray(FormArray array) {
    return array.controls.map((control) {
      if (control is FormGroup) return _serializeFormGroup(control);
      if (control is FormArray) return _serializeFormArray(control);
      if (control is FormControl) {
        final value = control.value;
        if (value is DateTime) {
          return {'_type': 'DateTime', 'value': value.toIso8601String()};
        }
        return value;
      }
      return null;
    }).toList();
  }

  static void _applyToFormGroup(FormGroup group, Map<String, dynamic> data) {
    for (final entry in data.entries) {
      if (!group.controls.containsKey(entry.key)) continue;

      final control = group.controls[entry.key]!;
      final value = entry.value;

      if (control is FormGroup && value is Map<String, dynamic>) {
        _applyToFormGroup(control, value);
      } else if (control is FormArray && value is List) {
        // Skip complex array restoration for simplicity
      } else if (control is FormControl) {
        if (value is Map && value['_type'] == 'DateTime') {
          control.value = DateTime.parse(value['value'] as String);
        } else {
          control.value = value;
        }
      }
    }
  }
}
```

---

## ขั้นตอนที่ 3128: Testing Form Validation

```dart
// test/form_validation_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:reactive_forms/reactive_forms.dart';
import 'package:advanced_forms_demo/validators/custom_validators.dart';

void main() {
  group('CustomValidators', () {
    test('strongPassword accepts valid password', () {
      final control = FormControl<String>(value: 'MyPass@123');
      final result = CustomValidators.strongPassword(control);
      expect(result, isNull);
    });

    test('strongPassword rejects weak password', () {
      final control = FormControl<String>(value: 'password');
      final result = CustomValidators.strongPassword(control);
      expect(result, isNotNull);
      expect(result!.containsKey('strongPassword'), isTrue);
    });

    test('passwordsMatch validates matching passwords', () {
      final form = FormGroup({
        'password': FormControl<String>(value: 'MyPass@123'),
        'confirmPassword': FormControl<String>(value: 'MyPass@123'),
      }, validators: [CustomValidators.passwordsMatch]);

      expect(form.valid, isTrue);
    });

    test('passwordsMatch fails on mismatch', () {
      final form = FormGroup({
        'password': FormControl<String>(value: 'MyPass@123'),
        'confirmPassword': FormControl<String>(value: 'different'),
      }, validators: [CustomValidators.passwordsMatch]);

      expect(form.hasError('passwordsMatch'), isTrue);
    });

    test('range validator works correctly', () {
      final validator = CustomValidators.range(18, 65);

      final validControl = FormControl<int>(value: 30);
      expect(validator(validControl), isNull);

      final tooLow = FormControl<int>(value: 10);
      expect(validator(tooLow), isNotNull);

      final tooHigh = FormControl<int>(value: 70);
      expect(validator(tooHigh), isNotNull);
    });
  });

  group('FormGroup integration', () {
    late FormGroup registrationForm;

    setUp(() {
      registrationForm = FormGroup({
        'username': FormControl<String>(
          validators: [Validators.required, Validators.minLength(4)],
        ),
        'email': FormControl<String>(
          validators: [Validators.required, Validators.email],
        ),
        'password': FormControl<String>(
          validators: [Validators.required, CustomValidators.strongPassword],
        ),
        'confirmPassword': FormControl<String>(
          validators: [Validators.required],
        ),
      }, validators: [CustomValidators.passwordsMatch]);
    });

    tearDown(() => registrationForm.dispose());

    test('form is invalid when empty', () {
      expect(registrationForm.invalid, isTrue);
    });

    test('form becomes valid with correct data', () {
      registrationForm.control('username').value = 'flutter_dev';
      registrationForm.control('email').value = 'test@example.com';
      registrationForm.control('password').value = 'SecurePass@123';
      registrationForm.control('confirmPassword').value = 'SecurePass@123';

      expect(registrationForm.valid, isTrue);
    });
  });
}
```

---

**← [Part 80](part-80-advanced-state.md)**
**ต่อไป: [Part 82 →](part-82-real-time-collaboration.md)**
